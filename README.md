# Домашнее задание: OSPF

## Текст задания

1. Поднять три виртуалки.
2. Объединить их разными VLAN.
3. Поднять OSPF между машинами на базе Quagga.
4. Изобразить ассиметричный роутинг.
5. Сделать один из линков «дорогим», но чтобы роутинг был симметричным.

Формат сдачи: Vagrantfile + ansible.

## Схема сети

```
        r1-r2 (10.0.10.0/30)
router1 ──────── router2 ────┐
   │                  │      │ r2-r3 (10.0.11.0/30)
   │  r1-r3           │      │
   └────── router3 ───┘      │
        (10.0.12.0/30) ──────┘

router1: 10.0.10.1 (r1-r2), 10.0.12.1 (r1-r3), 192.168.10.1/24 (net1)
router2: 10.0.10.2 (r1-r2), 10.0.11.2 (r2-r3), 192.168.20.1/24 (net2)
router3: 10.0.11.1 (r2-r3), 10.0.12.2 (r1-r3), 192.168.30.1/24 (net3)
```

Каждая пара роутеров соединена отдельной сетью/VLAN (в Vagrant — отдельные
`virtualbox__intnet`: `r1-r2`, `r2-r3`, `r1-r3`; на Proxmox-стенде — отдельные
изолированные бриджи vmbr-net11/12/13, изоляция эквивалентна VLAN).

## Реализация

- **FRR** (наследник Quagga, заброшен в 2018) — демоны `zebra` + `ospfd`, конфиг `/etc/frr/frr.conf`.
- router-id: 1.1.1.1 / 2.2.2.2 / 3.3.3.3, все сети в area 0.
- hello 10 / dead 30, `ip ospf mtu-ignore` на линковых интерфейсах.
- forwarding=1, `rp_filter=0` (нужно для ассиметричного роутинга).

Стенд развёрнут на **Proxmox** (ВМ 520 router1, 521 router2, 522 router3, Ubuntu 24.04),
артефакты для сдачи — Vagrantfile + ansible (эквивалентная конфигурация).

### 1. OSPF поднят

```
router1# show ip ospf neighbor
Neighbor ID     Pri State    Address     Interface
2.2.2.2           1 Full/DR  10.0.10.2   enp6s19
3.3.3.3           1 Full/DR  10.0.12.2   enp6s20

router1# show ip route ospf
O>* 10.0.11.0/30    [110/2] via 10.0.10.2
O>* 192.168.20.0/24 [110/2] via 10.0.10.2
O>* 192.168.30.0/24 [110/2] via 10.0.12.2
```

Пинги сквозные: router1 → 192.168.30.1 и router3 → 192.168.10.1 — 0% потерь.

### 2. Ассиметричный роутинг

`ip ospf cost 1000` на **router1, линк r1-r2 (enp6s19)**. Трафик r1 → net2
перестраивается через router3:

```
traceroute с router1 до 192.168.20.1:
 1  10.0.12.2   (router3)
 2  192.168.20.1
```

tcpdump на router2 подтверждает асимметрию:
- **enp6s20 (линк к router3)** — только входящие echo request от 192.168.10.1:
  `IP 192.168.10.1 > router2: ICMP echo request`
- **enp6s19 (прямой линк к router1)** — только исходящие echo reply:
  `IP router2 > 192.168.10.1: ICMP echo reply`

Туда — через router3, обратно — напрямую.

### 3. «Дорогой» линк + симметричный роутинг

Добавлен `ip ospf cost 1000` на **router2, тот же линк r1-r2 (enp6s19)**.
Теперь обратный трафик тоже идёт через router3 — роутинг симметричный:

```
traceroute с router2 до 192.168.10.1:
 1  10.0.11.1   (router3)
 2  192.168.10.1
```

tcpdump на router2 enp6s20 — request и reply в одном захвате:

```
IP 192.168.10.1 > router2: ICMP echo request
IP router2 > 192.168.10.1: ICMP echo reply
```

## Переключение ассиметрия ↔ симметрия

В `defaults/main.yml`: `symmetric_routing: false|true`. Применение без полного
прогона плейбука (теги):

```bash
ansible-playbook -i ansible/hosts -l all ansible/provision.yml -t setup_ospf \
  -e "symmetric_routing=true" -e "host_key_checking=false"
```

В шаблоне `frr.conf.j2`:

```jinja
{% if ansible_hostname == 'router1' %}
 ip ospf cost {{ expensive_cost }}
{% elif ansible_hostname == 'router2' and symmetric_routing == true %}
 ip ospf cost {{ expensive_cost }}
{% else %}
 !ip ospf cost {{ expensive_cost }}
{% endif %}
```

## Особенности и заметки

- Quagga мёртв (не развивается с 2018) — используется FRR, конфигурация совместима (vtysh, cisco-like).
- Для ассиметричного роутинга обязателен `rp_filter=0`, иначе ядро дропает «несимметричные» пакеты.
- Асимметрия строится дешёвым способом: cost 1000 на одном конце линка — прямой путь для r1 становится дороже обхода через r3, а у r2 обратный путь остаётся дешёвым.
- «Дорогой» линк остаётся рабочим (fallback): если r1-r3 падает, трафик вернётся на r1-r2 — OSPF перестроится автоматически.
- На Proxmox-стенде ВМ управлялись через `qm guest exec`; для ansible-пути у каждой ВМ есть mgmt-интерфейс (192.168.1.150-152) с дефолтом через хост-сеть.

## Файлы репозитория

| Файл | Назначение |
|---|---|
| `Vagrantfile` | 3 роутера + линковые сети + стаб-сети |
| `ansible/hosts` | инвентарь с router_id и адресами интерфейсов |
| `ansible/defaults/main.yml` | router_id_enable, symmetric_routing, cost |
| `ansible/templates/frr.conf.j2` | jinja2-шаблон OSPF-конфига с логикой cost |
| `ansible/templates/daemons` | включение zebra + ospfd |
| `ansible/provision.yml` | установка FRR, sysctl, конфиги, проверка соседей |
| `README.md` | этот файл |