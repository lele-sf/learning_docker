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
- **docker build -t <image_name> .**: cria uma imagem Docker a partir de um Dockerfile localizado no diretório atual. O parâmetro `-t` atribui um nome à imagem.
- **docker container run --name <container_name> <image_name>**: cria e inicia um novo container a partir de uma imagem especificada.
  - **-it**: combina `-i` (modo interativo) e `-t` (aloca um terminal), permitindo acesso interativo ao terminal do container.
  - **-d**: executa o container em segundo plano (modo "detached").
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

### Multistage Build no Dockerfile
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

## Glossário de Termos
- **kernel**: é o núcleo do sistema operacional, responsável por gerenciar os recursos do sistema e permitir a comunicação entre o hardware e o software.
- [**Docker Hub**](https://hub.docker.com/): repositório público onde usuários podem compartilhar e baixar imagens Docker.