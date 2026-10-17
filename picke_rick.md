# CTF write up: Pickle Rick

Write-up: Máquina Rick

Etapa 1: Reconhecimento inicial

Primeiro coloquei o endereço de IP no navegador e inspecionei a página para ver se tinha algo no HTML ou CSS. Tinha um comentário, como se o Rick tivesse anotado para ele mesmo lembrar:

```
R1ckRul3s
```

Etapa 2: Scan de portas

Abri o terminal e usei o IP da máquina (10.66.177.245):

bash

```bash
nmap 10.66.177.245
```

Verifiquei as portas abertas: a porta 22 e a 80 (HTTP).

Depois entendi que a porta 80 aberta é justamente o que indica que o site está hospedado e pode ser acessado via HTTP, então meio que pulei essa parte e ainda não entendi 100%.

Etapa 3: Enumeração de diretórios

Busquei usar o gobuster para ver diretórios, mas não consegui usar o /. Pesquisei e vi que procurar pelo robots.txt seria meio que igual:

bash

```bash
http://10.66.177.245/robots.txt
```

Encontrei a pista Wubbalubbadubdub. Também descobri que esse servidor roda Apache/2.4.41, não sei ainda se é útil.

Copiei e colei o / e fiz o mesmo para rodar o gobuster e procurar diretórios php e txt:

bash

```bash
gobuster dir -u http://10.66.177.245 -w /usr/share/wordlists/dirb/common.txt -x php,txt
```

O resultado revelou index.html, login.php e robots.txt.

Etapa 4: Acesso ao painel

Abri http://10.66.177.245/login.php e coloquei:

- Username: R1ckRul3s
- Senha: Wubbalubbadubdub

Consegui acessar o command panel.

Etapa 5: Exploração do painel

Vi em um vídeo que devia tentar usar algum comando tipo o cat para ver o conteúdo de arquivos. Primeiro dei ls e vi os arquivos. Rodei cat clue.txt e mostrou que tem proteção.

Nessa parte, por não saber, procurei a resposta e vi que grep . clue.txt funciona:

bash

```bash
grep . supersecret.txt
```

Achei o primeiro ingrediente: mr meeseek hair.

Usei grep . assets e não teve nada:

bash

```bash
grep . assets
```

Depois:

bash

```bash
less "/home/rick/second ingredients"
```

Achei o segundo: 1 jerry tear.

Por último:

bash

```bash
sudo less /root/3rd.txt
```

O último ingrediente era fleeb juice.

Etapa 6: Aprendizados

Pesquisei para que serve o gobuster e me ajudou a entender melhor: é uma ferramenta de enumeração de diretórios, arquivos e subdomínios em um servidor web (directory brute-forcing). Como as páginas costumam ter áreas que não aparecem nos links, o gobuster testa milhares de nomes de uma wordlist e mostra quais respondem com sucesso (200 OK).

Vi a definição de reverse shell:

A reverse shell is a technique where a compromised machine initiates a connection back to an attacker’s machine, giving the attacker remote command-line access to execute commands with the compromised process’s permissions.

Ainda não consigo fazer sozinho foi difícil