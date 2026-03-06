# monitoramentoipsecpfsense

# Manual – Monitoramento de Túneis IPsec pfSense no Zabbix

# Objetivo

Monitorar automaticamente:

- Status dos túneis IPsec
- Tráfego de cada túnel (IN/OUT)
- Descoberta automática de novos túneis
- Gráficos automáticos por túnel
- Integração com Grafana

Arquitetura final:

```
pfSense
   │
   │  (script + zabbix agent)
   │
Zabbix Server
   │
   ├─ Discovery de túneis
   ├─ Status VPN
   ├─ Tráfego IN/OUT
   └─ Gráficos automáticos
```

---

# 1. Pré-requisitos

Ambiente necessário:

- pfSense com IPsec configurado
- Zabbix Server funcionando
- Zabbix Agent instalado no pfSense
- Comunicação liberada na porta:

```
TCP 10050
```

Entre:

```
Zabbix Server → pfSense
```

---

# 2. Criar script para descobrir túneis IPsec

No pfSense crie o script:

```
/usr/local/bin/ipsec2_discover.sh
```

```
#!/bin/sh

echo'{ "data": ['

ipsec statusall |awk'
/^[[:space:]]*con[0-9]+\[/ {
  gsub("\\[.*","",$1)
  conn=$1
  getline
  name=$0
  sub(/^[[:space:]]+/,"",name)
  printf "{ \"{#CONN}\":\"%s\", \"{#NAME}\":\"%s\" },", conn, name
}
' |sed'$ s/,$//'

echo'] }'
```

Permissão:

```
chmod+x /usr/local/bin/ipsec2_discover.sh
```

Teste:

```
/usr/local/bin/ipsec2_discover.sh
```

Saída esperada:

```
{
 "data":[
  {"{#CONN}":"con10","{#NAME}":"AVAL/TOLEDO VPN"},
  {"{#CONN}":"con17","{#NAME}":"COBROACTIVO"}
 ]
}
```

---

# 3. Script para verificar status do túnel

Criar:

```
/usr/local/bin/ipsec_check.sh
```

```
#!/bin/sh

CONN=$1

ipsec statusall |grep-q"$CONN.*ESTABLISHED"

if [$?-eq0 ];then
 echo1
else
 echo0
fi
```

Permissão:

```
chmod +x /usr/local/bin/ipsec_check.sh
```

Teste:

```
/usr/local/bin/ipsec_check.sh con10
```

Retorno:

```
1 = UP
0 = DOWN
```

---

# 4. Script para coletar tráfego do túnel

Criar:

```
/usr/local/bin/ipsec_traffic.sh
```

```
#!/bin/sh

T="$1"
DIR="$2"

ipsec statusall2>/dev/null |awk-vt="$T"-vdir="$DIR"'
BEGIN { inb=0; outb=0 }

$0 ~ "^[[:space:]]*" t "_[0-9]+\\{" && $0 ~ /bytes_i/ {
    line=$0
    gsub(/,/, "", line)
    n=split(line, a, /[[:space:]]+/)

    for (i=1; i<=n; i++) {
        if (a[i] == "bytes_i") { v=a[i-1]; gsub(/[^0-9]/, "", v); inb += v }
        if (a[i] == "bytes_o") { v=a[i-1]; gsub(/[^0-9]/, "", v); outb += v }
    }
}

END {
  if (dir == "in")  { printf "%d\n", inb;  exit }
  if (dir == "out") { printf "%d\n", outb; exit }
}
'
```

Permissão:

```
chmod +x /usr/local/bin/ipsec_traffic.sh
```

Teste:

```
/usr/local/bin/ipsec_traffic.sh con10 in
/usr/local/bin/ipsec_traffic.sh con10 out
```

Retorno esperado:

```
35944289
111066704
```

---

# 5. Configurar UserParameter no Zabbix Agent

Editar:

```
/usr/local/etc/zabbix6/zabbix_agentd.conf
```

Adicionar:

```
UserParameter=ipsec2.discover,/usr/local/bin/ipsec2_discover.sh
UserParameter=ipsec2.tunnel[*],/usr/local/bin/ipsec_check.sh "$1"
UserParameter=ipsec2.traffic[*],/usr/local/bin/ipsec_traffic.sh "$1" "$2"
```

Reiniciar agent:

```
service zabbix_agentd restart
```

---

# 6. Testes no pfSense

Status túnel:

```
zabbix_agentd -t 'ipsec2.tunnel[con10]'
```

Tráfego:

```
zabbix_agentd -t 'ipsec2.traffic[con10,in]'
zabbix_agentd -t 'ipsec2.traffic[con10,out]'
```

---

# 7. Testes no Zabbix Server

Executar:

```
zabbix_get -s IP_PFSENSE -k ipsec2.discover
```

```
zabbix_get -s IP_PFSENSE -k 'ipsec2.traffic[con10,in]'
```

---

# 8. Criar Template no Zabbix

Criar template:

```
Template pfSense IPsec LLD
```

---

# 9. Criar Discovery Rule

Tipo:

```
Zabbix agent
```

Key:

```
ipsec2.discover
```

Intervalo:

```
1m
```

Macros:

```
{#CONN}
{#NAME}
```

---

# 10. Item Prototype – Status

Nome:

```
IPsec {#NAME} ({#CONN}) - Status
```

Key:

```
ipsec2.tunnel[{#CONN}]
```

Tipo:

```
Numeric unsigned
```

---

# 11. Item Prototype – Tráfego IN

Nome:

```
IPsec {#NAME} ({#CONN}) - IN
```

Key:

```
ipsec2.traffic[{#CONN},in]
```

Unidade:

```
Bps
```

Pré-processamento:

```
Modificações por segundo
```

---

# 12. Item Prototype – Tráfego OUT

Nome:

```
IPsec {#NAME} ({#CONN}) - OUT
```

Key:

```
ipsec2.traffic[{#CONN},out]
```

Unidade:

```
Bps
```

Pré-processamento:

```
Modificações por segundo
```

---

# 13. Trigger Prototype – VPN DOWN

Expressão:

```
min(/Template pfSense IPsec LLD/ipsec2.tunnel[{#CONN}],2m)=0
```

Nome:

```
IPsec {#NAME} DOWN
```

Severidade:

```
Alta
```

---

# 14. Graph Prototype

Nome:

```
IPsec {#NAME} ({#CONN}) - Tráfego
```

Itens:

```
IN
OUT
```

Resultado:

```
Gráfico automático por túnel
```

---

# 15. Resultado final no Zabbix

Para cada túnel:

```
IPsec AVAL/TOLEDO VPN (con10) - Status
IPsec AVAL/TOLEDO VPN (con10) - IN
IPsec AVAL/TOLEDO VPN (con10) - OUT
IPsec AVAL/TOLEDO VPN (con10) - Tráfego
```

Tudo criado automaticamente.

---

# 16. Integração com Grafana

No Grafana usar regex para pegar todos os túneis.

### Todos IN

```
/IPsec.*- IN/
```

### Todos OUT

```
/IPsec.*- OUT/
```

### Todos tráfegos

```
/IPsec.*-(IN|OUT)/
```

Evita pegar o item **Status**.

---

# 17. Benefícios da solução

✔ Descoberta automática de VPN

✔ Monitoramento de status

✔ Monitoramento de tráfego

✔ Gráficos automáticos

✔ Integração Grafana

✔ Escalável para centenas de túneis

Se amanhã criar:

```
con40
```

o Zabbix cria automaticamente:

```
Status
IN
OUT
Gráfico
Trigger
```

---

# 18. Estrutura final

```
pfSense
 ├─ ipsec2_discover.sh
 ├─ ipsec_check.sh
 └─ ipsec_traffic.sh

Zabbix
 ├─ LLD VPN
 ├─ Status
 ├─ Tráfego IN
 ├─ Tráfego OUT
 └─ Gráfico automático
```

---

Se quiser, posso também montar para você **a versão 2 desse manual**, com melhorias muito usadas em ambientes grandes:

- discovery automático de **Phase2**
- **top VPN tráfego**
- **VPN travada (UP sem tráfego)**
- **dashboard NOC de VPN**
- **mapa visual das VPN no Zabbix**.
