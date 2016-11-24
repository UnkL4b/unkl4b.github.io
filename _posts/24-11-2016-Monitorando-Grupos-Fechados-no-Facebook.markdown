---
layout: post
title:  "Monitorando Grupos Fechados no Facebook"
date:   2016-11-24
---

![img](https://media.giphy.com/media/GHycyakNPWSoo/giphy.gif)

####"Quis custodiet ipsos custodes?"

Pensei muito se escreveria esse post ou não, mas enfim, quero falar um pouco sobre monitoramento.
Em muitos trabalhos é preciso coletar informações, monitorar e ter a estratégia de como agir com determinado target, eu já tenho uma certa experiência com crawlers, mesmo que ainda seja pouca e me falte muito para realmente dizer que conheço sobre o assunto, aproveitando isso eu juntei o util com o agradável. 

Vivemos em uma época em que informações valem muito dinheiro, empresas podem prever ataques ou até mesmo fraudes por conta de uma informação relevantes.

Com base em um trabalho que estava realizando eu precisei monitorar algumas atitudes para proteger meu cliente, percebi que o principal meio de comunicação era grupos fechados de facebook, com isso eu procurei ferramentas em que eu pudesse monitorar esses tais grupos fechados ao público no facebook, não obtive sucesso na busca de uma ferramenta que fizesse isso e fui atrás de desenvolver algo para realizar o monitoramento.

Vou tentar passar a forma de criar um script básico para que você possa monitorar esse tipo de ação, apenas para ter uma noção de como eu fiz.

Alguns prós e contras do script:

#### Prós:
 - Você terá as postagens recentes dos grupos
 - Conseguirá prever uma ação coordenada 
 - Pode usar os dados obtidos para sua central de inteligência aprimorar o alcance do monitoramento

#### Contras:
 - Facebook não tem muitos recursos para esse tipo de monitoramento
 - Funciona como uma "gambiarra" de códigos
 - Breve esse recurso se tornará obsoleto
 - (IMPORTANTE) Terá que estar dentro do grupo no qual quer monitorar

Com base nos contras, dá para perceber que o script não é recomendado para um produto, sim para um monitoramento breve.

Bom, deixando a enrolação de lado, vamos partir para o código em si.

O facebook na área de desenvolvedor tem um recurso chamado Graph que pode ser melhor compreendido aqui (https://developers.facebook.com/docs/graph-api).

Com o graphs do facebook, conseguimos através da API deles ler postagens a publica-las, sabendo disso, fui pesquisa uma maneira de ler o conteúdo de grupos de facebook, descobri que da pra ler postagens em comunidades fechadas apenas nas APIs v2.1 a v2.3, atualmente a api do facebook está na v2.8, com isso não conseguiria fazer a pesquisa sem gerar um token, tive que fazer uma "gambiarra" usando selenium.

### Gerando token de acesso a API do Facebook

Antes de gerar o token, temos que entender o que vamos pesquisar e como pesquisar, o facebook disponibiliza via GET a api, uma consulta simples consiste no seguinte modo:

https://graph.facebook.com/v2.6/me?fields=id,name&access_token=**TOKEN_GERADO**

O resultado vem da seguinte forma em json(nessa consulta acima):

```json
{
id: "117488975323006",
name: "Danilo Vaz"
}
```

Beleza, entendo isso, vamos criar agora nosso token manualmente e depois montamos um script que gere o token.

Abra no navegador a pagina: https://developers.facebook.com/tools/explorer

![Token](https://unknownsec.files.wordpress.com/2016/10/captura-de-tela-2016-10-05-acc80s-17-24-03.png)

Notem que tem o token indicado como "**Token de acesso**", certo, veja também mais abaixo que indica a API usada, nesse caso a **2.8**.

Para fazer o que queremos temos que gerar o token na API **2.3**, ultima API com o recurso que vamos usar.

Clique em "Get Token"

![enter image description here](https://unknownsec.files.wordpress.com/2016/10/captura-de-tela-2016-10-05-acc80s-17-33-20.png)

Vai abrir uma janela igual a janela abaixo, em seguida clique no canto superior direito e selecione a **API 2.3**.

![enter image description here](https://unknownsec.files.wordpress.com/2016/10/captura-de-tela-2016-10-05-acc80s-17-33-35.png)

Agora vamos selecionar o item "**user_groups**" em destaque na imagem abaixo:

![](https://unknownsec.files.wordpress.com/2016/10/captura-de-tela-2016-10-05-acc80s-17-33-49.png)

Clique em "**Get Access Token**" que vai gerar o token que precisamos, agora, se você fizer novamente a consulta de um grupo privado ele vai retornar os posts do grupo.

![](https://unknownsec.files.wordpress.com/2016/10/captura-de-tela-2016-10-27-acc80s-00-24-24.png)

### Colocando em prática com python

Agora que já entendemos como gerar um token e extrair os posts, vamos ver o código pra realizar isso.

Eu fiz um commit com um script básico para entendermos.

https://github.com/UnkL4b/group-monitor

Eu separei ele em 4 scripts:

- facebook.py - script central, ele concentra o código pra gerar o token que vimos acima e realiza o get dos posts nos grupos.

- banco.py - faz o insert e cria o banco onde vamos armazenar os posts que coletamos

- genlog.py - gera logs de erros que possam dar, ele armazena um txt com a data do evento de log na pasta logs/ 

- realt.py - gera uma pagina HTML com os posts, sinceramente, não precisava, porém achei interessante uma mostragem dos resultados de forma simplificada

Vou explicar algumas partes do script que acho interessante, o resto, acho que fica facil de entender, vale a pena olhar o código e estuda-lo caso queira aprender.

#### Gerando token com o python

Pra gerar o token eu usei o selenium, basicamente ele simula um browser e realiza os processos humanos que fizemos manualmente para gera-lo de forma automática.

Outro detalhe, eu usei uma lib própria pra ralizar a consulta no facebook, achei mais simples fazer dessa forma que escrever o código realizando GET na api. A lib é a [facepy](https://github.com/jgorset/facepy), é bem fácil usa-la.

Voltando ao selenium, vou pular a parte de como baixa-lo, acredito que já deve saber como fazer, outra coisa, basta olhar o source.

Entre as linhas 107 e 146 está a classe responsável por gerar o token, ao estanciar o browser do selenium eu precisava acessar a pagina de login que redirecionava para a parte de DEV do facebook, pra isso eu fiz da seguinte forma:

    gen_browser = self.browser()
    gen_browser.get('https://www.facebook.com/login/?next=https%3A%2F%2Fdevelopers.facebook.com%2Ftools%2Fexplorer')

Repare na URL usada, ela redireciona direto para a tool explorer do facebook, mas antes disso, precisamos realizar o login no facebook, nas linhas 111 a 115 eu faço a procura pelo id da tag HTML, não foi muito difícil achar, olhando o código fonte facilmente você acha.

    set_email = gen_browser.find_element_by_id('email')
    set_email.send_keys(self.user_mail)
    set_senha = gen_browser.find_element_by_id('pass')
    set_senha.send_keys(self.user_pass)
    set_senha.send_keys(Keys.RETURN)

Primeiro eu crio a variável "set_email" com o elemento cuja ID seja 'email', esse é o text-box onde inserimos nosso e-mail ou telefone para login, logo na linha abaixo, eu envio para o campo como o parâmetro passado ao script com a nomenclatura '-u' de user, ele lê e digita no campo.

Fiz o mesmo processo para a senha, após digitar, envio um ENTER para a pagina, fazendo o login e me redirecionando para a pagina de dev.

Se olhar na linha 116 vai ver que coloco uma espera de 10 segundos, isso ajuda no tempo de carregamento da pagina, caso a internet esteja um pouco lenta, dá o tempo de carregar todos os próximos elementos da pagina a ser utilizados.

A partir da linha 117 eu faço o mesmo processo do login, porém pegando o ID de todos os elementos que clicamos e selecionamos no processo manual.

    get_token = gen_browser.find_elements_by_class_name('_55pe')
    get_token[1].click()
    print("[+] Etapa 1/6 concluída!")
    self.espera(5)
    get_useracc = gen_browser.find_element_by_class_name('_2nax')
    get_useracc.click()
    print("[+] Etapa 2/6 concluída!")
    self.espera(3)
    get_versao = gen_browser.find_elements_by_class_name('_55pe')
    get_versao[len(get_versao)-1].click()
    print("[+] Etapa 3/6 concluída!")
    self.espera(3)
    set_versao = gen_browser.find_element_by_link_text('v2.3')
    set_versao.click()
    print("[+] Etapa 4/6 concluída!")
    self.espera(2)
    set_user_group = gen_browser.find_element_by_name('user_groups')
    set_user_group.click()
    gen_token = gen_browser.find_elements_by_class_name('_4jy0')
    gen_token[len(gen_token)-3].click()
    print("[+] Etapa 5/6 concluída!")
    self.espera(2)
    token = gen_browser.find_elements_by_class_name('_58al')
    access_token = token[1].get_attribute('value')
    self.timestamp1 = datetime.datetime.now()
    gen_browser.close()
    print("[+] Etapa 6/6 concluída!")
    print("\n[+] Consultando grupos...")
    print("\n[+] Token: %s" % access_token)
    return access_token

Basicamente é assim que gero o token, como não tem como usa-lo no processo normal da API, tive que fazer essa "gambiarra".

Depois desse processo é apenas parser de JSON, acredito que vai conseguir entender facilmente o código.

Outro ponto, eu separei a pasta configs/ para colocar os grupos no qual quero monitorar, por exemplo:

Dentro da pasta tem um arquivo chamado 'config.json', a estrutura dele é a seguinte:

    {
        "facebook":[
                    {
                    "id":"782600765175254",
                    "id":"193890690988723",
                    "id":"1639686002956595",
                    "id":"877606639022820",
                    "id":"296686553788980",
                    "id":"764752213659675"
                    }
        ]
    }

Em cada ID é colocado o ID do grupo privado no qual o usuário que você vai usar para monitorar tem acesso, como disse no inicio do artigo, você precisa estar dentro do grupo para monitorar, como disse o Snowden, "Quando se está dentro se assume que é parte do sistema".

Para utilizar o script é simples, a syntax é a seguinte:

    ~$> python facebook.py -u "usuario" -p "senha"

Segue prints do script rodando:

![exec do script](https://unknownsec.files.wordpress.com/2016/11/captura-de-tela-2016-11-24-acc80s-18-37-04.png)

Basicamente é isso galera, queria mostrar que é possível criar um monitoramento de grupos fechados de facebook, embora seja um meio generico, mas é possível.
