# Giropops Senhas - Dockerização Otimizada

Esta pasta contém apenas os arquivos de Docker (`Dockerfile` e `docker-compose.yaml`) utilizados para o desafio de criar a imagem mais otimizada possível do projeto [giropops-senhas](https://github.com/badtuxx/giropops-senhas), proposto no curso Descomplicando Docker do LinuxTips.

## Sobre o Desafio

No curso, foi lançado o desafio de dockerizar o projeto giropops-senhas, buscando criar a imagem Docker mais enxuta e eficiente possível, sem incluir arquivos desnecessários e utilizando boas práticas de multi-stage build.

**Importante:**  
O código-fonte completo do projeto não está incluído aqui, pois pertence ao repositório original do [badtuxx/giropops-senhas](https://github.com/badtuxx/giropops-senhas). Aqui você encontra apenas a solução Docker.

## Estrutura dos Arquivos

- `Dockerfile`: Utiliza multi-stage build para criar um ambiente Python isolado e copiar apenas o necessário para a imagem final, reduzindo o tamanho e melhorando a segurança.
- `docker-compose.yaml`: Configuração para orquestrar a aplicação e o Redis.

## Técnicas de Otimização Utilizadas

- **Multi-stage build:**
    - Primeiro estágio para instalar dependências
    - Segundo estágio apenas com os componentes necessários para execução
- **Imagens Chainguard:**
    - Utiliza imagens Python minimalistas da Chainguard
    - Mais leves e seguras que as imagens Python padrão
- **Ambiente Virtual Python:**
    - Isolamento das dependências
    - Evita conflitos entre pacotes
- **Otimização de Camadas:**
    - Apenas os arquivos necessários são copiados para a imagem final

## Como Executar
Para executar este projeto:

```bash
docker compose up -d
```

A aplicação estará disponível em `http://localhost:5000`

## Detalhes da Implementação
- O Docker Compose configura dois serviços: a aplicação Flask e o Redis
- A aplicação se conecta ao Redis usando o hostname `redis` através da rede interna do Docker
- Não é necessário configurar o Redis manualmente, o Docker Compose gerencia tudo