# Descomplicando Docker

## O que é um container?
Container é o agrupamento de uma aplicação e suas dependências, que compartilham o mesmo kernel do sistema operacional do host (máquina física ou virtual).

## Máquina Virtual vs Container
Na máquina virtual você emula um novo sistema operacional dentro do sistema operacional do host. Já no container você emula somente as aplicações e suas dependências tornando o processo mais leve e rápido.

## Descomplicando Namespaces
Os namespaces são uma funcionalidade do kernel do Linux que permite isolar recursos entre diferentes processos. Isso significa que cada container tem seu próprio "enviroment", ou seja, cada container tem sua própria visão dos recursos do sistema, como processos, usuários, rede e sistemas de arquivos. Isso garante que os containers não interfiram uns nos outros, mesmo que estejam rodando no mesmo host.

### Exemplos de Namespaces
- **PID Namespace**: permite que cada container tenha seus próprios identificadores de processo (PID). Isso significa que, dentro do container, os processos parecem começar no PID 1 e estão isolados dos processos de outros containers. No entanto, o host ainda pode ver e gerenciar esses processos, que aparecem com PIDs diferentes no namespace do host.
- **Net Namespace**: permite que cada container possua sua própria interface de rede e portas, isolando o tráfego de rede entre containers. Para que seja possível a comunicação entre os containers, o Docker cria dois Net Namespaces diferentes:
  - Um Net Namespace para o container, que contém uma interface de rede (normalmente chamada de `eth0`).
  - Um Net Namespace para o host, que contém uma interface virtual chamada de `veth*` (onde `veth` é seguido por um identificador aleatório).
  - Essas duas interfaces (`eth0` no container e `veth*` no host) estão conectadas entre si, formando um par de interfaces virtuais. No host, essas interfaces são linkadas à bridge padrão do Docker (`docker0`), que atua como um switch virtual. A bridge permite que os containers se comuniquem entre si e com o host, roteando pacotes de rede de forma eficiente.
- **Mnt Namespace**: permite que cada container tenha seu próprio sistema de arquivos, isolando os arquivos e diretórios entre containers. Isso significa que cada container pode ter sua própria versão de bibliotecas e dependências, sem interferir nas versões instaladas no host ou em outros containers.
- **UTS Namespace**: responsável por isolar o hostname, nome de domínio, versão do kernel e outras informações relacionadas ao sistema.
- **User Namespace**: responsável por isolar os IDs de usuário e grupo, permitindo que os containers tenham seus próprios usuários e permissões.

### Copy-on-Write (CoW) e Docker

O conceito de **Copy-on-Write (CoW)** significa que um recurso, como um bloco no disco ou uma área na memória, só é copiado ou alocado quando ocorre uma modificação. Até então, o recurso original é compartilhado entre os processos ou containers, economizando espaço e recursos.

No contexto do Docker, o CoW é usado para gerenciar as camadas (layers) dos containers. Um container é composto por uma pilha de camadas, onde:

- As camadas inferiores são **read-only** (somente leitura) e compartilhadas entre vários containers.
- A camada superior é **read-write** (leitura e escrita), permitindo que o container faça alterações sem modificar as camadas originais.

Quando um arquivo ou recurso precisa ser alterado, ele é copiado da camada read-only para a camada read-write, onde as modificações são aplicadas. Isso torna o Docker eficiente, pois evita duplicação desnecessária de dados e permite que múltiplos containers compartilhem as mesmas camadas base.

## Comandos Docker
- **docker build -t <image_name> .**: cria uma imagem Docker a partir de um Dockerfile localizado no diretório atual.
  - **-t**: atribui um nome à imagem, no formato `repositorio:tag`. Se a tag não for especificada, o padrão é `latest`.
- **docker container run --name <container_name> <image_name>**: cria e inicia um novo container a partir de uma imagem especificada.
  - **-it**: combina `-i` (modo interativo) e `-t` (aloca um terminal), permitindo acesso interativo ao terminal do container.
  - **-d**: executa o container em segundo plano (modo "detached").
  - **-p <host_port>:<container_port>**: mapeia uma porta do host para uma porta do container, permitindo acesso externo ao serviço rodando no container.
- **docker container rm -f <container_name>**: remove um container, mesmo que ele esteja em execução. O parâmetro `-f` força a remoção.
- **docker container ls -a**: lista todos os containers, incluindo os que estão parados.
- **docker container start/stop/pause/unpause <container_name>**: inicia, para, pausa ou retoma um container em execução.
- **docker container attach <container_name/id>**: conecta o terminal do host ao terminal do container em execução.
- **docker container stats**: exibe informações em tempo real sobre o uso de recursos dos containers em execução, como CPU, memória e rede.
- **docker container top <container_name/id>**: exibe os processos em execução dentro de um container específico, semelhante ao comando `top` do Linux.
- **docker container logs --details <container_name/id>**: exibe os logs de um container, incluindo detalhes adicionais, como timestamps e IDs de contêiner.
- **docker container exec -it <container_name/id> <command>**: executa um comando dentro de um container em execução, permitindo interagir com o ambiente do container.
- **docker pull <image_name>**: baixa uma imagem do Docker Hub ou de um repositório remoto.

### Observação sobre o uso do terminal interativo (-it)
Se você está rodando um container com `-it` e o processo principal é o `bash`, não use `exit` para sair, pois isso encerra o container. Use `Ctrl + P + Q` para sair do terminal e deixar o container rodando em segundo plano. Depois, use `docker ps` para confirmar que ele continua em execução.

## Explicando o Dockerfile
O Dockerfile abaixo serve para criar uma imagem personalizada baseada no Ubuntu 18.04 com o Nginx instalado e pronto para uso:

```dockerfile
FROM ubuntu:18.04
LABEL maintainer="email@gmail.com"
RUN apt-get update && apt-get install nginx -y
EXPOSE 80
ADD node_exporter-1.9.1.darwin-amd64.tar.gz /root/node_exporter
COPY index.html /var/www/html/
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
WORKDIR /var/www/html
VOLUME /var/www/html
ENV APP_VERSION=1.0.0
HEALTHCHECK --timeout=2s CMD curl -f localhost || exit 1
```
- **FROM**: especifica a imagem base a ser utilizada (neste caso, o Ubuntu 18.04).
- **LABEL**: adiciona metadados à imagem, como o autor e a descrição.
- **RUN**: executa comandos no momento da construção da imagem. Aqui, atualiza os repositórios e instala o Nginx.
- **EXPOSE**: informa ao Docker que a aplicação dentro do container escutará na porta 80.
- **ADD**: adiciona o arquivo `node_exporter.tar.gz` ao diretório `/root/node_exporter` dentro do container. O comando `ADD` também pode descompactar arquivos tar automaticamente. E se o arquivo for um URL, ele será baixado.
- **COPY**: copia o arquivo `index.html` do diretório atual do host para o diretório `/var/www/html/` dentro do container.
- **ENTRYPOINT**: define o comando que será executado quando o container for iniciado. Neste caso, inicia o Nginx.
- **CMD**: fornece argumentos adicionais para o comando definido em `ENTRYPOINT`. Aqui, o Nginx é configurado para rodar em primeiro plano (`daemon off`).
- **WORKDIR**: define o diretório de trabalho dentro do container. Todos os comandos subsequentes serão executados a partir deste diretório.
- **VOLUME**: cria um ponto de montagem para persistência de dados. Neste caso, o diretório `/var/www/html` é montado como um volume, permitindo que os dados sejam preservados mesmo se o container for removido.
- **ENV**: define uma variável de ambiente dentro do container, neste caso, `APP_VERSION` com o valor `1.0.0`.
- **HEALTHCHECK**: define um comando para verificar a saúde do container. Neste caso, verifica se o serviço está respondendo na porta 80 usando `curl`. Se o comando falhar, o container é considerado não saudável.

### CMD vs ENTRYPOINT
Ambas as instruções definem o que é executado quando um container inicia, mas funcionam de maneira diferente:

- **CMD**:
  - Define comandos e/ou argumentos padrão que são facilmente substituídos
  - Quando você executa `docker run imagem [arg]`, o argumento substitui completamente o CMD
  - Usado para definir comportamentos padrão que podem ser alterados

- **ENTRYPOINT**:
  - Define o executável principal que sempre será executado
  - Não é facilmente substituído (requer flag `--entrypoint`)
  - Garante que uma aplicação específica seja sempre executada

- **Uso combinado**: quando ambos estão presentes, o ENTRYPOINT define o executável e o CMD fornece os argumentos padrão.

**Exemplo**:
```dockerfile
ENTRYPOINT ["nginx"]  # executável fixo
CMD ["-g", "daemon off;"]  # args que podem ser substituídos
```

- Se executar `docker run imagem` → executa `nginx -g daemon off;`
- Se executar `docker run imagem -v` → executa `nginx -v`

## Multistage Build no Dockerfile
O **Multistage Build** é uma técnica no Docker que permite criar imagens mais eficientes e leves, dividindo o processo de construção em várias etapas.

Isso é especialmente útil para projetos que requerem compilação, como aplicações em Go, Java ou Node.js. A ideia é usar uma imagem temporária para compilar o código e, em seguida, copiar apenas os artefatos necessários para a imagem final.

**Como funciona**: 
O multistage permite usar `múltiplas instruções FROM` no mesmo Dockerfile, cada uma representando uma etapa de construção. A imagem final é construída a partir da última etapa, mas pode copiar arquivos de etapas anteriores.

**Exemplo**:
```dockerfile
# Etapa de construção
FROM golang:1.20 AS builder
WORKDIR /app
COPY . ./
RUN go mod init hello
RUN go build -o /app/hello

# Etapa final
FROM alpine:latest
COPY --from=builder /app/hello /app/hello
CMD ["/app/hello"]
``` 
Neste exemplo, a primeira etapa usa a imagem do Go para compilar o código, enquanto a segunda etapa usa uma imagem mais leve (Alpine) para criar a imagem final, contendo apenas o binário compilado. Isso reduz significativamente o tamanho da imagem final, pois não inclui todas as dependências de compilação.

## Imagens Distroless

Imagens **Distroless** são imagens de container que não incluem uma distribuição Linux completa, apenas o mínimo necessário para rodar a aplicação (runtime e dependências). Elas não possuem shell, gerenciadores de pacotes ou utilitários do sistema operacional.

**Vantagens:**
- Imagens muito menores
- Menos vulnerabilidades de segurança (menos componentes expostos)
- Deploys mais rápidos

**Desvantagens:**
- Não possuem shell ou ferramentas para depuração dentro do container
- Exigem mais atenção às dependências da aplicação

**Como usar:**  
Você pode usar imagens distroless como base no seu Dockerfile, por exemplo:

```dockerfile
FROM golang:1.20 AS build
WORKDIR /app
COPY . .
RUN go build -o hello

FROM gcr.io/distroless/base
COPY --from=build /app/hello /
CMD ["/hello"]
```

### Implementando Distroless

Existem várias maneiras de adotar uma estratégia Distroless. As mais comuns são utilizar imagens base já prontas, como:

- [Google Distroless](https://github.com/GoogleContainerTools/distroless): imagens minimalistas mantidas pelo Google, muito usadas para aplicações em Go, Java, Node.js, Python, entre outras.
- [Chainguard Images](https://edu.chainguard.dev/chainguard/chainguard-images/about/getting-started-distroless/): imagens modernas, seguras e também minimalistas, com versões para várias linguagens (ex: Python, Node, Java).

Essas imagens já vêm preparadas para produção, sem shell ou utilitários extras, e são ideais para quem busca máxima segurança e eficiência.

## Scanner de Vulnerabilidades em Imagens

Garantir a segurança das imagens de container é fundamental, já que vulnerabilidades presentes nas camadas ou pacotes podem comprometer toda a aplicação. Para isso, existem ferramentas especializadas que analisam imagens e apontam possíveis falhas de segurança.

### Docker Scout

O [**Docker Scout**](https://docs.docker.com/scout/) é uma ferramenta oficial do Docker para análise de vulnerabilidades em imagens. Ele gera um inventário completo dos pacotes (SBOM - Software Bill of Materials) e compara com bancos de dados de vulnerabilidades atualizados. O Scout pode ser usado via Docker Desktop, Docker Hub, linha de comando (`docker scout cves`), ou integrado a pipelines CI/CD. Ele mostra CVEs, recomendações de correção e permite comparar imagens.

### Trivy

O [**Trivy**](https://trivy.dev/latest/getting-started/) é uma ferramenta open source muito popular para escanear vulnerabilidades em imagens de container, arquivos, repositórios de código e até configurações de infraestrutura como código. Ele é simples de usar, rápido e pode ser integrado facilmente em pipelines de CI/CD.

Exemplo de uso do Trivy:
```sh
trivy image nome-da-imagem
```
O Trivy também suporta análise de arquivos Dockerfile, repositórios Git e diretórios locais.

## Cosign e a Importância de Assinar Imagens

Garantir a integridade e a autoria das imagens de container é fundamental para a segurança em ambientes de produção. Ferramentas como o [**Cosign**](https://docs.sigstore.dev/cosign/overview/) permitem **assinar digitalmente** imagens de container, comprovando que elas não foram alteradas e que realmente foram criadas por quem afirma tê-las criado.

**Por que assinar imagens?**
- Evita o uso de imagens adulteradas ou maliciosas.
- Permite verificar a procedência da imagem antes do deploy.
- Facilita auditorias e conformidade em ambientes corporativos.

**Como funciona o Cosign?**
O Cosign gera uma assinatura digital para a imagem e armazena essa assinatura no próprio registro de imagens (como Docker Hub, GitHub Container Registry, etc). Antes de usar ou implantar uma imagem, é possível verificar a assinatura para garantir sua autenticidade.

**Exemplo básico de uso:**
```sh
# Gerar um par de chaves
cosign generate-key-pair

# Assinar uma imagem
cosign sign --key cosign.key usuario/imagem:tag

# Verificar a assinatura
cosign verify --key cosign.pub usuario/imagem:tag
```

O uso de assinaturas digitais em imagens é uma prática recomendada para aumentar a segurança da cadeia de suprimentos de software (supply chain security) e evitar ataques como o "supply chain attack".

## Volumes no Docker

Volumes são mecanismos para persistir e compartilhar dados entre containers ou entre containers e o host. Eles resolvem um problema fundamental: o sistema de arquivos dos containers é **efêmero** - quando um container é removido, todos os dados não persistidos são perdidos.

### Por que usar volumes?

- **Persistência de dados**: banco de dados, uploads, configurações e logs são preservados mesmo quando o container é destruído
- **Compartilhamento**: facilita a troca de dados entre containers ou entre host e container
- **Performance**: em alguns casos, volumes oferecem melhor desempenho do que a escrita no sistema de arquivos do container
- **Portabilidade**: volumes gerenciados pelo Docker funcionam em qualquer ambiente Docker

### Tipos de montagem para persistência de dados

O Docker suporta dois tipos principais de montagem para persistência de dados:

#### 1. Bind Mounts

Monta um arquivo ou diretório específico do host diretamente no container.

**Características:**
- Liga diretamente um caminho do host a um caminho no container
- Ideal para desenvolvimento quando precisa compartilhar código-fonte
- Depende da estrutura de diretórios do host
- Permite acesso completo a arquivos do host a partir do container

**Exemplos:**

Com `--mount` (sintaxe explícita e recomendada):
```sh
# Monte um diretório do host no container
docker container run -ti --mount type=bind,src=/home/user/projeto,target=/app ubuntu

# Montagem somente leitura (read-only)
docker container run -ti --mount type=bind,src=/home/user/configs,target=/etc/configs,ro ubuntu
```

Com `-v` (sintaxe abreviada):
```sh
docker run -v /home/user/projeto:/app ubuntu
```

#### 2. Volumes Docker

Volumes são áreas de armazenamento gerenciadas pelo Docker, independentes da estrutura de diretórios do host.

**Características:**
- Gerenciados pelo Docker (armazenados em `/var/lib/docker/volumes/`)
- Independentes da estrutura do host (mais portáveis)
- Podem ser facilmente copiados, compartilhados e migrados
- Suportam drivers de armazenamento para funcionamento com sistemas de armazenamento remoto
- Recomendados para uso em produção

**Gerenciando volumes:**

```sh
# Criar um volume
docker volume create app_data

# Listar volumes
docker volume ls

# Inspecionar detalhes
docker volume inspect app_data

# Remover um volume
docker volume rm app_data

# Remover volumes não utilizados
docker volume prune
```

**Montando volumes:**

Com `--mount` (sintaxe explícita):
```sh
docker container run -d --mount type=volume,source=app_data,target=/data nginx
```

Com `-v` (sintaxe abreviada):
```sh
docker container run -d -v app_data:/data nginx
```

Se o volume não existir, o Docker o cria automaticamente quando solicitado.

### Comparação: Bind Mounts vs Volumes Docker

| Característica | Bind Mounts | Volumes Docker |
|----------------|-------------|---------------|
| **Localização** | Qualquer lugar no host | `/var/lib/docker/volumes/` |
| **Gerenciamento** | Manual | Automático pelo Docker |
| **Ideal para** | Desenvolvimento, compartilhamento de arquivos | Produção, persistência de dados |
| **Portabilidade** | Baixa (depende da estrutura do host) | Alta (independente do host) |
| **Compartilhamento** | Entre host e containers | Entre containers |
| **Segurança** | Pode expor arquivos do sistema | Mais isolado e seguro |

## Redes no Docker

As redes no Docker permitem que containers se comuniquem entre si e com o mundo externo. Por padrão, o Docker cria uma rede isolada para cada container, mas você pode configurar redes personalizadas para controlar como os containers interagem.

### Por que usar redes customizadas?

- **Isolamento**: separe containers de diferentes aplicações em redes distintas
- **Comunicação segura**: controle quais containers podem se comunicar
- **Descoberta de serviços**: containers podem se comunicar usando nomes ao invés de IPs
- **Flexibilidade**: configure diferentes topologias de rede conforme a necessidade

### Tipos de rede no Docker

#### 1. Bridge (Default)

A rede padrão que conecta containers no mesmo host através de uma bridge virtual.

**O que é uma bridge?**
Uma bridge é como um **switch de rede virtual** criado pelo Docker. Tipo um switch físico onde você conecta vários dispositivos, a bridge funciona da mesma forma mas virtualmente, conectando containers entre si e permitindo que se comuniquem.

**Características:**
- Rede padrão para containers standalone
- Containers podem se comunicar por IP (na rede bridge padrão) ou por nome (em redes customizadas)
- Acesso externo via mapeamento de portas (`-p`)
- Isolamento entre containers de diferentes redes bridge

**Exemplo:**
```sh
# Container usa a rede bridge padrão
docker run -d --name web nginx

# Container com rede bridge customizada
docker network create my-network
docker run -d --name app --network my-network nginx
```

#### 2. Host

Remove o isolamento de rede entre o container e o host Docker.

**Características:**
- Container compartilha a pilha de rede do host
- Melhor performance (sem overhead de bridge)
- Menos isolamento (container vê todas as interfaces do host)
- Disponível apenas em hosts Linux

**Exemplo:**
```sh
docker run -d --name app --network host nginx
```

#### 3. None

Remove completamente a interface de rede do container.

**Características:**
- Container sem conectividade de rede externa
- Máximo isolamento
- Útil para containers que só processam dados locais

**Exemplo:**
```sh
docker run -d --name isolated --network none alpine sleep 1000
```

#### 4. Overlay

Conecta containers executando em diferentes hosts Docker, criando uma rede virtual distribuída.

**Características:**
- Usado em Docker Swarm ou clusters
- Permite comunicação entre containers em hosts diferentes
- Criptografia de tráfego automática
- Descoberta de serviços integrada (containers encontram uns aos outros automaticamente)

**Exemplo:**
```sh
# Requer Docker Swarm iniciado
docker network create --driver overlay --attachable my-overlay
```

### Resumo: Quando usar cada tipo de rede

| Tipo | Alcance | Caso de uso principal |
|------|---------|---------------------|
| **Bridge** | **Mesmo host** | Containers no mesmo servidor se comunicando |
| **Host** | **Mesmo host** | Máxima performance, compartilha rede do host |
| **None** | **Isolado** | Sem conectividade de rede |
| **Overlay** | **Múltiplos hosts** | Containers em servidores diferentes se comunicando |

### Comandos básicos para gerenciar redes

```sh
# Listar redes
docker network ls

# Criar uma rede
docker network create my-network

# Inspecionar rede
docker network inspect my-network

# Conectar container a uma rede
docker network connect my-network my-container

# Desconectar container de uma rede
docker network disconnect my-network my-container

# Remover rede
docker network rm my-network

# Remover redes não utilizadas
docker network prune
```

## Docker Compose

Docker Compose é uma ferramenta para definir e executar aplicações Docker multi-container. Com um único arquivo YAML, você configura todos os serviços da sua aplicação e, com um único comando, cria e inicia todos os containers definidos.

### Por que usar Docker Compose?

- **Simplificação**: defina toda a sua aplicação em um único arquivo declarativo
- **Consistência**: garante que todos os componentes sejam executados com as configurações corretas
- **Facilidade de manutenção**: alterações em sua infraestrutura ficam versionadas como código
- **Automação**: inicie e pare múltiplos containers com um único comando
- **Reutilização**: compartilhe composições entre ambientes de desenvolvimento, teste e produção

### Componentes principais do Docker Compose

#### 1. Arquivo docker-compose.yaml

O arquivo `docker-compose.yaml` é o coração do Docker Compose. Ele define todos os serviços, redes, volumes e outras configurações necessárias para sua aplicação.

**Estrutura básica**:
```yaml
services:        # Definição dos containers/serviços
networks:        # Redes personalizadas
volumes:         # Volumes persistentes
```

#### 2. Definição de Serviços

Cada serviço representa um container que será executado como parte da sua aplicação.

**Exemplo básico**:
```yaml
services:
  webapp:                     # Nome do serviço
    image: nginx:latest       # Imagem a ser usada
    ports:
      - "8080:80"             # Mapeamento de porta
  database:
    image: mysql:8.0          # Outro serviço na mesma composição
    environment:
      MYSQL_ROOT_PASSWORD: secret
```

### Principais recursos do Docker Compose

#### 1. Builds personalizados

Em vez de usar uma imagem existente, você pode construir uma imagem personalizada:

```yaml
services:
  webapp:
    build:                    # Em vez de image:
      context: ./app          # Diretório contendo o Dockerfile
      dockerfile: Dockerfile  # Nome do arquivo (opcional se for o padrão)
      args:                   # Argumentos para o build
        VERSION: '1.0'
```

#### 2. Redes

Crie redes personalizadas para isolamento e comunicação entre serviços:

```yaml
services:
  webapp:
    networks:
      - frontend
  api:
    networks:
      - frontend
      - backend
  database:
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
```

#### 3. Volumes

Volumes persistentes para armazenamento de dados:

```yaml
services:
  database:
    volumes:
      - db-data:/var/lib/mysql    # Volume nomeado
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # Bind mount

volumes:
  db-data:    # Definição do volume nomeado
```

#### 4. Variáveis de ambiente e arquivos

Configure serviços com variáveis de ambiente:

```yaml
services:
  webapp:
    environment:              # Variáveis inline
      NODE_ENV: production
      API_URL: http://api:3000
    env_file:                 # Ou de um arquivo
      - ./config.env
```

#### 5. Dependências entre serviços

Defina a ordem de inicialização dos serviços:

```yaml
services:
  webapp:
    depends_on:
      - api
      - database
```

#### 6. Configuração de recursos e políticas de restart

Controle de recursos e comportamento em caso de falha:

```yaml
services:
  webapp:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
      restart_policy:
        condition: on-failure
        max_attempts: 3
```

### Comandos básicos do Docker Compose

```sh
# Iniciar todos os serviços definidos no docker-compose.yaml
docker compose up

# Iniciar em modo detached (background)
docker compose up -d

# Parar todos os serviços
docker compose down

# Ver logs de todos os serviços
docker compose logs

# Ver status dos serviços
docker compose ps

# Reiniciar um serviço específico
docker compose restart api

# Executar um comando em um serviço
docker compose exec api bash

# Construir ou reconstruir serviços
docker compose build

# Escalar um serviço para múltiplas instâncias
docker compose up -d --scale api=3
```

## Glossário de Termos
- **kernel**: é o núcleo do sistema operacional, responsável por gerenciar os recursos do sistema e permitir a comunicação entre o hardware e o software.
- [**Docker Hub**](https://hub.docker.com/): repositório público onde usuários podem compartilhar e baixar imagens Docker.