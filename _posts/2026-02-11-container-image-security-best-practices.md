---
title: "Segurança e boas práticas em imagens de container"
date: 2026-02-11 09:00:00 -0300
categories: security
tags: [container, segurança, imagem, docker, boas-praticas]
description: "Nesse artigo vamos ver algumas maneiras de deixar a nossa imagem mais segura em nossos containers."
image: /assets/img/container-image-security-best-practices.png
---

Nos dias de hoje temos um vasto ecossistema de container que é utilizado seja em um cluster kubernetes, seja em um docker swarm (sim algumas pessoas ainda usam em produção =/ ), ou em serviços lock-in como o ECS. 

Porém muita das vezes quando falamos de segurança pensamos em diversos fatores com longas discussões para manter o ambiente seguro, envolvemos equipes de desenvolvedores, devops, network e etc. 

É discutido ideias mirabolantes de permissionamento, camadas de firewall, RBAC, policy, ajuste no cloud onde a aplicação esta rodando, caso esteja em alguma cloud, mas por muita das vezes esquecemos o básico ou talvez a primeira coisa a se olhar que é onde sua aplicação realmente ira ficar que é a imagem que ira ficar dentro de um container.

Sei que é estranho falar isso, mas sim as vezes criamos fortalezas com diversas camadas de segurança e esquecemos de olhar para algo teoricamente básico e simples que esta la no começo do processo, e eu vejo isso como o seguinte, é como se criássemos uma fortaleza “intransponível”, com proteções de todos os lados e de todas as formas, mas não cuidássemos do que esta entrando pela porta da frente, seria como se a fortaleza fosse um banco “super seguro” mas não tivesse um detector de metais na entrada, ou não tivesse nenhum guarda para verificar quem esta entrando, na fortaleza super segura.

E é mais ou menos isso que fazemos quando não damos a devida atenção para nossas imagens que colocamos em nossos containers. Devido a isso vamos abordar algumas coisinhas que vão ajudar muito a deixar tudo mais seguro.

## ***O que é Dockerfile e Imagem de Container ?***

Antes de mais nada vamos entender o que é uma imagem de container e o que é um dockerfile, para que possamos entender melhor o que estamos falando.

***Dockerfile*** = É um arquivo de texto simples que contém uma série de instruções que o Docker utiliza para construir uma imagem de container. Ele define o ambiente, as dependências, as configurações e os comandos necessários para criar uma imagem que pode ser executada como um container. Você pode imaginar o dockerfile como uma receita que descreve quais são os passos que o docker vai seguir para criar a imagem desejada, colocando configurações personalizadas de acordo com a necessidade da aplicação.

***Imagem de Container*** = Uma imagem de container é um conjunto de componentes que foram empacotados para ser executado em um ambiente de container. Em nosso caso ela vai ser construida a partir do dockerfile, essa imagem é composta pos algumas camadas, e cada camada tem um papel importante para o funcionamento e para o desempenho da aplicação, e também para a sua segurança.



* _Imagem base ou a golden image_ = Seria o S.O minimo ali que você precisa para executar algo

* _Bibliotecas e Binarios_ = Executaveis e dependencias essenciais para as coisas acontecerem

* _Dependencias da aplicação_ = Pacotes especificos que são exigidos pela aplicação como um npm, pip e etc

* _Configurações e metadados_ = Variaveis de ambiente, portas, comandos de entrada e etc, são configurações passadas para aquela particularidade.


* ### ***Montando nosso dockerfile***

Agora que sabemos o que basicamente se compoe uma imagem e o que é um dockerfile, vamos criar um exemplo simples para que possamos analisar a segurança da imagem e depois melhorar a mesma.

Mas antes precisamos criar a nossa aplicação, e para isso vamos criar algo bem simples em Go, precisamos então criar alguns arquivos, o primeiro deles é o main.go, que vai ser a nossa aplicação, e o segundo é o go.mod que é o arquivo de configuração do nosso projeto em Go, para que possamos baixar as dependencias da aplicação.

Crie esses arquivos dentro do diretorio ``` /src  ```

Arquivo main.go:

***main.go***

```
package main

import (
        "fmt"
        "log"
        "net/http"
        "os"
)

func main() {
        port := os.Getenv("PORT")
        if port == "" {
                port = "8080"
        }

        http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {

                fmt.Fprintf(w, "Olá galerinha acessem https://blog.rafael.bhz.br/")
        })

        log.Printf("Servidor rodando na porta %s...", port)
        if err := http.ListenAndServe(":"+port, nil); err != nil {
                log.Fatal(err)
        }
}
```

Arquivo go.mod:

***go.mod***

```
module github.com/rafaelfernandessilva

go 1.22.2
```

Vamos criar nosso dockerfile um diretorio acima.

***Dockerfile***

```Dockerfile
FROM golang:1.22

WORKDIR /app

COPY src/go.mod .
COPY src/main.go ./


RUN go build -o main .

CMD ["./main"]
```

* ### ***Fazendo o build da imagem***

Com os arquivos da aplicação criado e nosso dockerfile criado, vamos fazer o build da imagem para que possamos analisar a mesma e ver se tem vulnerabilidades e depois melhorar.

Para isso vamos executar o comando de build do docker.

> docker build -t app-go:1.0 .

Nesse comando estamos chamando do docker passando como pametro o build, depois o -t para dar um nome para a imagem, no caso app-go:1.0 e por fim o ponto que indica que o dockerfile esta no diretorio atual.

Pronto agora já temos o build da nossa primeira imagem.

Veja que foi feito o build da imagem e podemos ver algumas coisas sobre ela, vamos primeiro ver o tamanho da mesma com o comando:

> ***docker images -f 'reference=app-go:1.0'***

![size image](/assets/img/size_image_2.png)

Veja que uma aplicação super simples ficou com mais de 1gb de tamanho. O que não é tão interessante para esse caso visto que temos muita coisa sem necessidade nessa imagem e como e algo compilado não estamos usando boas pratica e isso não esta diretamente ligado a segurança mas já que estamos otimizando nossa imagem temos que levar isso em consideração.

## ***Realizando scan e detectando vulnerabilidades***
E para realmente fazer uma analise da segurança da nossa imagem vamos utilizar duas ferramentas extremamente poderosas, que é o docker scout e o trivy.

Ta beleza vamos usar essas ferramentas, mas de onde vem? O que são ? de que se alimentam ?

* ### ***Docker Scout***
O Docker Scout é uma solução focada na segurança proativa da cadeia de suprimentos de software. Ele funciona analisando imagens de containers para identificar e mitigar vulnerabilidades antes que elas se tornem problemas críticos. As principais funcionalidades são:

***Análise e SBOM:*** O Docker Scout analisa suas imagens e compila um inventário detalhado de componentes, chamado de Software Bill of Materials (SBOM).

***Detecção de Vulnerabilidades:*** Esse SBOM é comparado com um banco de dados de vulnerabilidades que é atualizado continuamente, permitindo localizar falhas de segurança específicas nos pacotes e camadas da imagem.

* ### ***Trivy***

O Trivy é uma ferramenta de segurança abrangente e versátil, amplamente utilizada para escanear não apenas imagens de containers, mas também diversos outros artefatos em busca de falhas de segurança, é uma ferramenta da Aqua Security e diferente do docker scout que é atrelada ao docker o trivy é uma ferramenta de segurança a parte. As principais funcionalidades do trivy:

***Detecção de Múltiplas Ameaças:*** Ele varre as imagens em busca de vulnerabilidades conhecidas (CVEs), configurações incorretas (como erros em arquivos de infraestrutura como código), segredos expostos (como chaves de API e senhas) e licenças de software.

***Alvos de Varredura:*** O Trivy analisa tanto os arquivos internos da imagem quanto os seus metadados/configurações (como as instruções do Dockerfile), identificando, por exemplo, se um container está configurado para rodar como usuário root.

***Gestão de SBOM:*** Assim como as soluções modernas de segurança, o Trivy suporta a geração e a descoberta de SBOMs (Software Bill of Materials), permitindo inventariar todos os componentes de uma imagem para uma análise mais rápida e precisa.


Agora que conhecemos duas ótimas ferramentas para nos ajudar na segurança da nossa imagem vamos então ver como está a nossa situação.

Vamos primeiro utilizar o docker scout para ver se estamos com muitas vulnerabilidades, para isso partindo do principio que temos docker e o docker scout vamos utilizar o comando:


> ***docker scout quickview app-go:1.0***


O comando docker scout com o parametro quickview vai nos da um overview resumido sobre como esta nossa situação em relação a imagem, que estamos construindo. Nossa saída vai ser algo semelhante a isso:

![docker scout quickview](/assets/img/docker_quickview.png)

Temos também outro comando muito importante O comando docker scout cves que nos vai dar uma visão completa e detalhada de todas as vulnerabilidades (CVEs) identificadas em uma imagem de container.

> ***docker scout cves app-go:1.0***

Nessa primeira temos o que foi encontrado: 

![docker scout cves](/assets/img/docker_scout_1.png)

Podemos também utilizar o trivy, para analisar a situação da nossa imagem:


> ***docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy \
 image --severity HIGH,CRITICAL,MEDIUM app-go:1.0***


![trivy scan](/assets/img/trivy-1.png)


Como vimos o resultado não foi muito bom, temos diversas vulnerabilidades de nível CRITICAL, e isso é algo que devemos nos preocupar, pois essas vulnerabilidades podem ser exploradas e comprometer toda a nossa aplicação e infraestrutura.

## ***Otimizando nossa imagem e elimando vulnerabilidades***

Para melhorar a segurança da nossa imagem, vamos realizar alguns procedimentos, o primeiro deles é atualizar a versão do golang que estamos utilizando, pois como vimos em ambos os scan tivemos problema com a versão do golang da nossa aplicação então vamos primeiro atualização a mesma pois é uma das coisas indicadas como fix.

Nossa primeira alteração vai ser buscar a imagem com versão mais recente do golang então altere para ```FROM golang:1.25```, e faça novo builda da imagem.

Para isso vamos alterar o nosso dockerfile para:

```dockerfile   
FROM golang:1.25
WORKDIR /app

COPY src/go.mod .
COPY src/main.go ./


RUN go build -o main .

CMD ["./main"]
```

Vamos fazer um novo build da imagem com o comando:

> docker build -t app-go:2.0 .

Observe que agora estamos utilizando a versão 1.25 do golang, e estamos fazendo o build da imagem com o nome app-go:2.0 para que possamos diferenciar da imagem anterior.

Feito isso vamos rodar novamente o nosso scan, na nova imagem que fizemos o build no meu caso a app-go:2.0

> docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --severity HIGH,CRITICAL,MEDIUM app-go:2.0 

Agora já temos um cenario bem melhor: 

![trivy scan](/assets/img/trivy-2.png)

Docker scout: 

> ***docker scout cves app-go:2.0***

![docker scout cves](/assets/img/docker_scout_2.png)

Já não temos alerta de nivel CRITICAL, mas temos uns outros no qual ainda devemos nos preocupar que é gerado devido ao S.O base no qual a imagem esta.

Então vamos dar um jeito nisso, e para isso vamos usar a imagem da chainguard, eles tem com foco ter imagens totalmente seguras sem cve, e também vamos usar uma imagem distroless com as boas práticas de mult stage no nosso dockefile, pois como nossa aplicação é compilada, precisamos apenas do arquivo gerado para podermos executar a mesma.

Vamos fazer uma alteração no nosso arquivo Dockerfile:

Dockerfile

```dockerfile

FROM cgr.dev/chainguard/go:latest AS builder

WORKDIR /app

COPY src/go.mod ./
RUN go mod download

COPY src/main.go ./

RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o main .

FROM gcr.io/distroless/static-debian12

COPY --from=builder /app/main /main

USER nonroot:nonroot

ENTRYPOINT ["/main"]
```
Nessa alteração do dockerfile estamos realizar o multi stage. A pratica de utilizar o multi stage é uma ótima maneira de otimizar a imagem final, pois no primeiro estágio no qual damos o nome de builder, estamos compilando a nossa aplicação realizando o download das dependencias e demais procedimentos necessarios, e no segundo estágio estamos apenas pegando o que é necessario para rodar a aplicação, que é o arquivo main gerado no primeiro estagio. 

Com esse novo dockerfile vamos fazer um novo build da imagem:

> docker build -t app-go:3.0 .

Agora criando uma imagem com o nome app-go:3.0 para que possamos diferenciar das imagens anteriores, e depois vamos rodar novamente o scan para ver como esta a situação da nossa imagem.

* ***Trivy***
> docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --severity HIGH,CRITICAL,MEDIUM app-go:3.0 


![trivy scan](/assets/img/trivy-3.png)


* ***Docker scout***

> ***docker scout cves app-go:3.0***

![docker scout cves](/assets/img/docker_scout_3.png)

Agora temos uma imagem mais segura que podemos utilizar em nossa infra estrutura. E ainda tem mais, a nossa imagem agora esta bem menor. Se executarmos o comando:

{% raw %}

> docker images app-go:3.0 --format '{{.Size}}'

{% endraw %}

![size image](/assets/img/size_image.png)

Vamos ver que nossa imagem inicial tinha mais 1GB e nossa imagem final tem menos de 20MB.

Com isso acho que já sabemos o basico sobre como melhorar a segurança em nossos contrainer e talvez já podemos dizer se somos o melhor da nossa rua talvez.


Referência: 
- [Site Oficial do Trivy](https://trivy.dev/docs/latest/guide/target/container_image/)
- [Aqua Vulnerability Database (AVD)](https://avd.aquasec.com/)
- [Diretório de Imagens Chainguard](https://images.chainguard.dev/)
- [Docker Scout](https://docs.docker.com/engine/scan/scout/)
- [SBOM: Software Bill of Materials](https://www.ntia.gov/software-bill-materials)
- [CVEs: Common Vulnerabilities and Exposures](https://cve.mitre.org/)
- [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Docker CLI](https://docs.docker.com/reference/cli/docker/)