---
layout: post
title:  "Formatando saida do NMAP com Python"
date:   2016-05-11
---
# Formatando saida do NMAP com Python

![img](https://media.giphy.com/media/K09XMn8LNleEg/giphy.gif)

Olá pessoas do meu blog, tudo bem com vocês?

Estou sumido não é mesmo?

Bom, faz algumas semanas que não abro o bash nem minha console python para brincar, eu tenho trabalhado e estudado demais, por conta do trabalho terei que tirar algumas certificações, mas eu já estava com vontade escrever sobre o que vou falar aqui para vocês. A algumas semanas atrás eu comecei a estudar uma lib em python que usa o NMAP para realizar scans, nada novo, ela apenas usa o NMAP como um subprocess e formata a saída do resultado.

Ai vocês me perguntam:
 
 -- "Ok seu animal, então pq você vai nos mostrar algo que já existe, quer reinventar a roda?"

Não meus amigos, na verdade até vou, mas vou dar um caminho diferente.

O pq de usar uma lib e programar algo que já existe, eu vou explicar, simples. Em um pentest geralmente precisamos de resultados específicos para gerar um relatório ou até mesmo para uma filtragem melhor de serviço e com o python temos essa possibilidade.

Primeiro, vamos instalar a lib necessária para a criação de um script personalizado do nmap.

```sh
$ pip install python-nmap
```

Feito isso, vamos começar entendendo como a lib funciona.

Apenas uma observação, não vou mostrar como o NMAP funciona, caso queira entender sobre o nmap, aconselho a digitar o comando abaixo:

```sh
$ man nmap
```

Embora para você montar algum script com essa lib você tenha que ter o mínimo de noção de nmap, eu não vou explicar aqui, apenas o básico para dar um "NORTE" para o uso da lib.

Voltando, a lib como já disse acima, usa o nmap em um subprocess, se você abrir a lib python-nmap, pode observar na linha 63 a importação do subprocess e na linha 228 a utilização do nmap através do subprocess. (Considerando a ultima atualização de '2016.03.15' as linhas permanecem as mesmas)

LINHA:**63**
```python
63 import subprocess
```

LINHA: **228 - 231**
```python
228 p = subprocess.Popen(args, bufsize=100000,
229                     stdin=subprocess.PIPE,
230                     stdout=subprocess.PIPE,
231                     stderr=subprocess.PIPE)
```

Ainda olhando na lib, veremos que ele faz um parser no resultado em XML do nmap, gerado pelo argumento "-oX", como pode ser visto na linha 224.

```python
224 args = [self._nmap_path, '-oX', '-'] + h_args + ['-p', ports]*(ports is not None) + f_args
```

Então, essa lib poderia ser feita ou melhorada por qualquer um de nós, vale ressaltar aqui que muitos(me incluo nessa) tem medo de colaborar com projetos assim, vamos perder esse medo em galera!

Entendendo isso sabemos que TEMOS que ter o nmap instalado certo amiguinhos?

Vamos começar a brincar um pouco.

Todo o resultado do nmap será carregado em um dict na variável que você setar, vamos ver como fica, vamos fazer um scan básico.

Outro detalhe importante, estou usando python2.7, porque? Não sei, mas estou :P

Não testei no python3.x

```python
import nmap

nm = nmap.PortScanner()
scan = nm.scan(hosts="127.0.0.1",arguments="-sS -sU -p 80")

print(scan)
```

O resultado será igual a esse:

```
{'nmap': {'scanstats': {'uphosts': '1', 'timestr': 'Wed May 11 01:33:40 2016', 'downhosts': '0', 'totalhosts': '1', 'elapsed': '0.51'}, 'scaninfo': {'udp': {'services': '80', 'method': 'udp'}, 'tcp': {'services': '80', 'method': 'syn'}}, 'command_line': 'nmap -oX - -sS -sU -p 80 127.0.0.1'}, 'scan': {'127.0.0.1': {'status': {'state': 'up', 'reason': 'localhost-response'}, 'udp': {80: {'product': '', 'state': 'closed', 'version': '', 'name': 'http', 'conf': '3', 'extrainfo': '', 'reason': 'port-unreach', 'cpe': ''}}, 'vendor': {}, 'addresses': {'ipv4': '127.0.0.1'}, 'tcp': {80: {'product': '', 'state': 'closed', 'version': '', 'name': 'http', 'conf': '3', 'extrainfo': '', 'reason': 'reset', 'cpe': ''}}, 'hostnames': [{'type': 'PTR', 'name': 'localhost'}]}}}
```

Dando aquele tapa, arrumei em formato json para melhor visualização:

```json
{
  "nmap": {
    "scanstats": {
      "uphosts": "1",
      "timestr": "Wed May 11 01:33:40 2016",
      "downhosts": "0",
      "totalhosts": "1",
      "elapsed": "0.51"
    },
    "scaninfo": {
      "udp": {
        "services": "80",
        "method": "udp"
      },
      "tcp": {
        "services": "80",
        "method": "syn"
      }
    },
    "command_line": "nmap -oX - -sS -sU -p 80 127.0.0.1"
  },
  "scan": {
    "127.0.0.1": {
      "status": {
        "state": "up",
        "reason": "localhost-response"
      },
      "udp": {
        "80": {
          "product": "",
          "state": "closed",
          "version": "",
          "name": "http",
          "conf": "3",
          "extrainfo": "",
          "reason": "port-unreach",
          "cpe": ""
        }
      },
      "vendor": {},
      "addresses": {
        "ipv4": "127.0.0.1"
      },
      "tcp": {
        "80": {
          "product": "",
          "state": "closed",
          "version": "",
          "name": "http",
          "conf": "3",
          "extrainfo": "",
          "reason": "reset",
          "cpe": ""
        }
      },
      "hostnames": [
        {
          "type": "PTR",
          "name": "localhost"
        }
      ]
    }
  }
}
```

Em um simples scan podemos observar o que vem, já conseguimos realizar uns filtros aqui, por exemplo, podemos executar algo expecífico caso encontremos a porta 80 aberta, o bom de receber o valor dentro do python é a possíbilidade de criar infinitas possibilidades de exploração em um pentest.

Eu não vou prolongar muito nesse post, pretendo abordar melhor nos próximos posts, mas como uma intro, fiz um script para exemplificar o uso do nmap e vocês verem o que podemos fazer.

```python
import nmap

nm = nmap.PortScanner()
nm.scan(hosts='ig.com.br', arguments='-sS -p 80')
for hst in nm.all_hosts():
    print hst
    if nm[hst].has_tcp(80) is True:
        import requests
        url = "http://%s" % hst
        r = requests.get(url)
        print(r.text)
```

No script acima, assim que encontramos uma porta 80 aberta, realizamos um GET na pagina e printamos o conteúdo, isso poderia ser utilizado juntamente com o script NSE do nmap de verificação de métodos do servidor e caso encontre o método PUT já realize um upload de um arquivo para o servidor alvo, em um ambiente de pentest onde se tem muitos sevidores é um bom adianto para automatizar o pentest.

Bom, fica a dica galera, vou aprofundar mais na lib do nmap nos próximos posts, agora estou com sono e tenho que acordar cedo amanha, abraço!!!

Já ia me esquecer, vou palestrar na BSIDES LATAM 2016 que ocorre nos dias 11 e 12 na PUC-SP Consolação, para maiores informações, podem consultor esse link: http://latam.securitybsides.com.br/

Eu vou apresentar a pesquisa que fiz do GitMiner, quero ver vocês lá!!!
