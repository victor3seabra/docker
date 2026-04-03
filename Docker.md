Se trata de uma ferramenta de containerização de aplicações

Com Docker podemos fazer deploy de Containers que poderão ser executados em diversos lugares sem a necessidade de se adequar a especificações de cada máquina
# Arquitetura
# Imagens
São a planta baixa da definição de como o Container deve operar

Definimos uma imagem de acordo com as especificações que desejamos num arquivo denominado Dockerfile

Podemos criar um Container a partir de uma imagem ao executar `docker build .`

Podemos importar um imagem já existente num registry e executá-la, gerando o Container, usando `docker run`

A customização de imagens é comum pois, frequentemente gostaríamos de adicionar arquivos e/ou funcionalidades ao Container já em sua criação, como por exemplo instalação de um pacote Go

**Imagens são imutáveis**
## Layers
A estrutura de um Dockerfile é construída em camadas (layers) onde **cada layer é uma instrução a ser realizada na criação do Container**

![](images/image-1.png)

**Podemos reutilizar layers ja existentes em outras imagens**, tornando o desenvolvimento mais ágil.

![](images/image-2.png)
## Dockerfile
Se trata do documento que define as instruções (Layers) que serão usadas para a criação de uma Imagem. Para tal podemos fazer uso dos seguintes comandos:
### FROM
Define a imagem base a ser usada, bem como sua versão e tags

É a primeira Layer, podendo ser precedida somente por ARG
### WORKDIR
Define o diretório onde as subseguintes Layers irão operar
### COPY
Realiza a cópia de dados de uma fonte para um destino sendo o destino a localização no Container
### ADD
Adiciona arquivos remotos ou locais com a possiblidade de, por exemplo, descomprimir um arquivo .tar
### RUN
Realiza a execução de um dado comando durante o processo de build da imagem
### ENV
Permite setar variaveis de ambiente a serem usadas pelo Container

Podemos mudar as variaveis de ambiente usando a flag -e na notação `docker run -e foo=bar postgres env`

Podemos restringir recursos do container fazendo uso de --memory e --cpus como `docker run -e POSTGRES_PASSWORD=secret --memory="512m" --cpus="0.5" postgres`
### ARG
Permite definir variáveis, invisíveis ao Container, mas úteis no desenvolvimento pois torna o código mais DRY
### CMD
Determina o comando padrão a ser executado quando um Container estiver sob execução
### ENTRYPOINT
Especifica um executável padrão a ser utilizado na inicialização do container
### USER
Define um perfil de usuário que irá executar as subseguintes etapas
### EXPOSE
Define a porta que o Container deve expor para se comunicar na ordem HOST:CONTAINER quando passado a flag -p no CLI

Podemos omitir a porta do host em situações onde queremos somente mudar a porta efemera do container seguindo a notação
```docker run -p 80 nginx```

Podemos publicar em todas as portas efemeras com o intuito de evitar conflitos de uso de portas usando a flag -P
### HEALTHCHECK
Faz a checagem da saúde do container em sua inicialização
### LABEL
Nos permite inserir metadados à imagem
### MAINTAINER
Especifica o autor da imagem
### ONBUILD
### SHELL
Define a shell padrão de uma imagem
### STOPSIGNAL
### Parser directives
**São opcionais e afetam como as linhas seguintes serão lidadas no Dockerfile mas sem adicionar novas Layers**

Tem como sintaxe `# directive=value` sendo as directives possíveis:
* `syntax` (não é case sensitive)
    * especifíca a versão da sintaxe a ser seguida no build `# syntax=docker/dockerfile:1`. se nao especificado usa a versão do Docker instalado na maquina
* `escape` 
    * especifíca caracteres de escape para, por exemplo, quebra de linha. se não especificado usa \
* `check` (não é case sensitive e seus valores devem fazer uso de Pascal case)
    * define como a checagem do build das Layers serão feitas, por padrão, todos os checks são feitos e falhas tratadas como Warnings
    * podemos por exemplo ignorar alguns checks feitos por padrão `# check=skip=JSONArgsRecommended,StageNameCasing` ou até mesmo desativar todos os checks `# check=skip=all`
    * **podemos fazer builds quebrarem na presença de Warnings usando `# check=error=true`**, quando usado essa Parser directive é de bom tom definirmos a `syntax` pois em versões futuras da imagem podemos ter problemas inesperados
    * **para incluirmos mais de uma checagem devemos separá-las por ; `# check=skip=JSONArgsRecommended;error=true`**

**Após um comentário, linha vazia ou Layer o Parser directive não é mais considerado, logo, nosso Parser directive deve estar no inicio do Dockerfile**

**É convenção deixar linhas em branco entre Parser directives**

**Cada Parser directive deve ser único** sendo o exemplo abaixo inválido
```docker
# directive=value1
# directive=value2

FROM ImageName
```

Exemplo de caso inválido pois precede uma Layer:
```docker
FROM ImageName
# directive=value
```
## Cache
Docker se beneficia de cache de builds anteriores mitigando a necessidade de executar novamente o mesmo comando durante o novo build, tornando mais eficiente este processo

O Docker não irá fazer uso do cache em situações como:
1. Alterações de uma Layer RUN
2. Alterações em arquivos em COPY/ADD

**No momento que uma Layer tem seu cache invalidado, as subseguintes também o terão**
## Multi-stage builds
Possibilita a execução concorrente de etapas em diferentes ambientes tornando o build mais eficiente e, ao fim de cada uma dessas etapas, podemos selecionar somente o que nos é pertinente para a execução do Container, tornando-o mais leve e diminuindo a superfície de ataques.

Exemplo de aplicação Python:
```docker
# ── Stage 1: Builder ──────────────────────────────────────────
FROM python:3.12-slim AS builder

WORKDIR /app

# Install build dependencies
RUN pip install --upgrade pip
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ── Stage 2: Runtime ──────────────────────────────────────────
FROM python:3.12-slim AS runtime

WORKDIR /app

# Copy only installed packages from builder
COPY --from=builder /install /usr/local

# Copy application source
COPY src/ .

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

EXPOSE 8000
CMD ["python", "main.py"]
```

Exemplo de aplicação Go:
```docker
# ── Stage 1: Builder ──────────────────────────────────────────
FROM golang:1.22-alpine AS builder

WORKDIR /app

# Cache dependencies
COPY go.mod go.sum ./
RUN go mod download

# Build a statically linked binary
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o server ./cmd/server

# ── Stage 2: Runtime ──────────────────────────────────────────
FROM scratch AS runtime

# Optional: add CA certs if making HTTPS calls
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy only the binary
COPY --from=builder /app/server /server

EXPOSE 8080
ENTRYPOINT ["/server"]
```
# Containers
Se tratam de processos isolados que desejamos operar e que são definidos por uma Imagem que **não necessitam de um kernel, hardware, programas e aplicações do host pois possuem suas proprias**

Algumas características de Containers são:

* Cada Container tem tudo que precisa para desempenhar sua função

* São isolados de hosts e outros Containers, garantindo segurança

* Cada Container é gerenciado de forma independente

* Garantem reprodutibilidade 
# Volumes
São formas persistentes de armazenamento de dados de Containers dentro do Docker que podem ser usados por diversos containers

Podemos fazer sua criação via Dockerfile ou ainda via CLI da seguinte forma `docker volume create log-data`

Para que o container faça uso desse volume devemos atribuir a ele via CLI com a flag -v `docker run -d -p 80:80 -v log-data:/logs docker/welcome-to-docker`. **se o volume nao existir ele é criado automaticamente**
# Bind Mounts
São formas persistentes de armazenamento de dados de Containers no Host, isto é, na maquina em que está sendo executado o Container fazendo com que esses recursos sejam compartilhados em tempo real com o container

Podem ser criados usando a flag de volumes -v ou ainda --mount durante o docker run

O uso de -v faz com que seja criado um bind mount simples sem tanto controle granular quanto --mount `docker run -v /HOST/PATH:/CONTAINER/PATH -it nginx`

Com o uso de --mount podemos especificar, por exemplo, permissao somente de leitura `docker run -v HOST-DIRECTORY:/CONTAINER-DIRECTORY:ro nginx` (rw para read-write)

* ro nao permite alteração no host, somente leitura
* rw permite alteração no host, leitura e escrita

**Com --mount se o volume não existir ele nao é criado automaticamente**
# Namespaces
# overlayfs
