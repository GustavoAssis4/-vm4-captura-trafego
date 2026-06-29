# VM4 - Captura e Análise de Tráfego

## Objetivo

A VM4 é responsável pela captura passiva do tráfego espelhado da infraestrutura OpenStack, realizando a geração de fluxos de comunicação (Flow Generator) e enviando essas informações para a API REST hospedada na VM1.

---

# Topologia

A VM4 está conectada à VLAN de monitoramento (VLAN 40) e recebe o tráfego espelhado pelos switches virtuais.

Interfaces físicas:

- ens3 → Switch Virtual 1
- ens7 → Switch Virtual 2

Subinterfaces monitoradas pelo Flow Generator:

```
ens3.10
ens3.20
ens3.30
ens3.40
ens3.50
ens3.60

ens7.10
ens7.20
ens7.30
ens7.40
ens7.50
ens7.60
```

MAC Address da interface ligada ao SW1

```
fa:16:3e:77:99:5e
```

MAC Address da interface ligada ao SW2

```
fa:16:3e:69:d6:31
```

---

# Configuração inicial

Verificar interfaces

```bash
ip addr
```

Verificar rotas

```bash
ip route
```

Testar conectividade

```bash
ping 10.10.1.2
```

---

# Atualização do sistema

```bash
apt update
apt upgrade -y
```

---

# Instalação do Python

```bash
apt install python3 python3-pip -y
```

---

# Instalação das bibliotecas

```bash
pip install scapy requests
```

ou

```bash
pip install -r requirements.txt
```

---

# Resolução de DNS

Caso a VM não consiga acessar a internet, editar:

```bash
nano /etc/resolv.conf
```

Adicionar:

```
nameserver 8.8.8.8
nameserver 1.1.1.1
```

---

# Docker

Instalação

```bash
apt install docker.io -y
```

Verificação

```bash
docker --version
```

Construção da imagem

```bash
docker build -t flow-generator .
```

Listar imagens

```bash
docker images
```

Executar container

```bash
docker run -d \
--name flow-generator \
--net=host \
--privileged \
--log-opt max-size=20m \
--log-opt max-file=3 \
flow-generator
```

Verificar containers

```bash
docker ps
```

Ver logs

```bash
docker logs flow-generator
```

Entrar no container

```bash
docker exec -it flow-generator bash
```

---

# Execução sem Docker

```bash
python3 flow_generator.py
```

---

# Funcionamento do Flow Generator

A aplicação realiza continuamente as seguintes etapas:

1. Captura pacotes utilizando a biblioteca Scapy.
2. Identifica automaticamente a VLAN através da interface de captura.
3. Agrupa os pacotes em fluxos.
4. Calcula:
   - quantidade de pacotes;
   - quantidade de bytes;
   - duração;
   - taxa de transmissão;
   - protocolo;
   - serviço associado.
5. Exporta todos os fluxos para o arquivo:

```
flows.json
```

6. Envia apenas os fluxos novos para a API REST da VM1 utilizando requisições HTTP POST.
7. Remove automaticamente fluxos inativos após 60 segundos.

---

# API REST

Servidor:

```
VM1
```

Endpoint utilizado:

```
http://10.0.20.10/api/v1/flows/batch
```

Método

```
POST
```

Formato enviado

```json
[
    {
        "src_ip":"10.0.10.2",
        "dst_ip":"10.0.20.2",
        "src_port":50502,
        "dst_port":80,
        "protocol":"TCP",
        "bytes":1024,
        "packets":8,
        "duration_s":0.53
    }
]
```

---

# Estrutura do projeto

```
VM4/
│
├── Dockerfile
├── requirements.txt
├── flow_generator.py
├── flows.json
└── README.md
```

---

# Observações

- A VM4 trabalha exclusivamente de forma passiva.
- O modo promíscuo das interfaces é configurado pelo módulo responsável pelos switches virtuais (VM5).
- O Flow Generator apenas processa os pacotes recebidos pelo espelhamento de tráfego.
- Os logs do Docker possuem limite de tamanho para evitar esgotamento do disco da máquina virtual.
