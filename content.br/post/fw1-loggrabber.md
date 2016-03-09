+++
date = "2016-03-05T14:23:22+01:00"
title = "Coletando logs do Check Point usando fw1-loggrabber"
Description = "Extraindo logs para arquivos"

translationurl = "/post/fw1-loggrabber/"

tags = [
	"Check Point"
]
+++

## TL;DR
Coletar logs de firewalls Check Point é complicado. Nesse post, eu descrevo como consegui usar a ferramenta fw1-loggrabber rodando em um servidor Linux 32bit, capturar os logs de um Check Point Gaia R77.30, armazená-los em um arquivo e usar o logrotate para gerenciá-los.

## Intro
Check Point é um produto sensacional. Até que se queira coletar logs dele com uma ferramenta externa. No meu caso, eu preciso coletar seus logs para um servidor Linux para que eu possa fazer umas coisas legais com eles. [Existem](https://www.fir3net.com/Firewalls/Check-Point/a-quick-guide-to-checkpoints-opsec-lea.html) [vários](https://supportcenter.checkpoint.com/supportcenter/portal?eventSubmit_doGoviewsolutiondetails=&solutionid=sk32521) [métodos](https://blog.rootshell.be/2014/08/28/check-point-firewall-logs-and-logstash-elk-integration/) [disponíveis](http://www-01.ibm.com/support/knowledgecenter/SSSN2Y_1.0.0/com.ibm.itcim.doc/tcim85_install190.html%23opconz) em como fazer isso. Boa sorte com todos esses. Nenhum desses me ajudou a fazer o que eu fiz aqui nesse post. Por isso que é importante entender que esse guia usa o Check Point Gaia R77.30. O hardware pode ser qualquer coisa desde uma VM para um appliance. É possível que isso funcione em outros ambientes e outras versão. Ou não.

Para coletar logs do Check Point, o fw1-loggrabber será usado. Ele está [disponível aqui](https://github.com/certego/fw1-loggrabber). Pode-se também usar a versão [disponível no sourceforge](http://sourceforge.net/projects/fw1-loggrabber/) mas esse código é antigo. Pode-se ver o trabalho feito no código da versão do github. Essa ferramenta só roda em Linux x86 This tool only runs on Linux x86 running kernel 3.16, so a Linux Ubuntu x86 was installed in the Lab Server running at 172.16.1.10 com kernel 3.16. Ele também roda com kernel 4. A confiança (trust) do OPSEC LEA tem que ser configurado entre a ferramenta e o Check Point. A compilação do código do fw1-loggrabber não vai funcionar se todas as dependências do Linux estiverem satisfeitas.

Esse guia assume que o laboratório já está preparado com:

1. Um Check Point Gaia R77.30 com função Gateway and Management. No meu caso, é uma VM rodando os dois. O endereço IP é 172.16.1.1.
2. Um cliente Windows 7 para rodar o Check Point SmartDashboard. O endereço IP é 172.16.1.11.
3. Um servidor Linux x86 rodando kernel 3.16 para hospedar o fw1-loggrabber. O endereço IP é  172.16.1.10.

O processo de criação do laboratório é por sua conta. Se você quiser mais informação, me mande uma mensagem.

## Configurando o Check Point
Primeiramente, é preciso configurar a Manager do Check Point para aceitar a conexão OPSEC LEA. Isso pode ser feito assim:

1. Conecte-se ao SmartDashboard como admin

2. Crie um objeto Host para o servidor Linux que vai coletar os logs:  
	{{< figure src="/img/01.png" >}}
	O servidor Linux é 172.16.1.10:  
	{{< figure src="/img/02.png" >}}

3. Crie o objeto OPSEC:
	* Vá no "Servers and OPSEC" e clique com o botão direito no diretório "OPSEC Application". Crie um "New OPSEC Application":  
	{{< figure src="/img/03.png" >}}
	* Configure o objeto OPSEC. Dê um nome, selecione o Host que foi acabado de ser criado para o servidor Linux e selecione a opção LEA:     
	{{< figure src="/img/04.png" >}}  
	* Clique em Communication e crie uma nova senha SIC. Note que quando isso for feito, o acesso já estará disponível, sem a necessidade de se instalar a política no firewall:  
	{{< figure src="/img/05.png" >}}  

4. Talvez seja necessário adicionar uma regra de firewall para permitir a conexão OPSEC LEA entre o servidor Linux e a Manager Check Point usando porta TCP/18184 (FW1_lea)
	* De 172.16.1.10 para 172.16.1.1 na porta TCP/18184
 
## Configurando o Linux
Eu estou usando um Debian 8.2 no laboratório:
{{< figure src="/img/06.png" >}}  
A ferramenta fw1-loggrabber precisa ser compilada do código fonte. Vamos usar o usuário `user` e o diretório de trabalho `/home/user/fw1-loggrabber`.

1. Antes de começar, as seguintes dependências precisam ser instaladas. Isso irá provavelmente instalar mais inúmeros pacotes, mas isso vai depender de qual Linux você está usando:  
	```
	user@debian:~$ apt-get install git gcc-multilib g++-multilib libelf-dev:i386
	```

2. Faça o download da ferramenta:  
	```
	user@debian:~$ mkdir fw1-loggrabber; cd fw1-loggrabber
	user@debian:~/fw1-loggrabber$ git clone https://github.com/certego/fw1-loggrabber
	```  
	Isso fará com que o código-fonte do fw1-loggrabber seja armazenado em `/home/user/fw1-loggrabber/fw1-loggrabber`. Quando tudo estiver pronto, esse diretório poderá ser removido.

3. Baixe as bibliotecas de desenvolvimento necessários do OPSEC [do site da Check Point aqui](supportcontent.checkpoint.com/file_download?id=7385). Copie o arquivo `OPSEC_SDK_6.0_Linux.zip` para o diretório `/home/user/fw1-loggrabber` e o descompresse. Isso é necessário para compilação do fw1-loggrabber.  
	```
	user@debian:~/fw1-loggrabber$ unzip OPSEC_SDK_6.0_Linux.zip
	```  
	Esse arquivo zip contém o SDK linux no arquivo `OPSEC_SDK_6_0.linux30.tar.gz`. Esse arquivo precisa ser descomprimido no mesmo diretório do fw1-loggrabber com o seguinte comando:  
	```
	user@debian:~/fw1-loggrabber$ tar zxvf OPSEC_SDK_6_0.linux30.tar.gz
	```  
	Os arquivos serão extraídos no diretório `pkg_rel`.

4. Compile o fw1-loggrabber. Mas primeiro, vá em `/home/user/fw1-loggrabber/fw1-loggrabber` e troque a seguinte linha no arquivo `Makefile` de acordo com esse modelo:  
	```
	PKG_DIR = ../pkg_rel
	```  
	Então finalmente compile o código e o instale para que seja possível utilizá-lo posteriormente:  
	```
	user@debian:~/fw1-loggrabber/fw1-loggrabber$ make
	```  
	{{< figure src="/img/07.png" >}}  
	```
	user@debian:~/fw1-loggrabber/fw1-loggrabber$ sudo make install
	```  
	{{< figure src="/img/08.png" >}}  
	O fw1-loggrabber será instalado em `/usr/local/fw1-loggrabber/`. Quando isso estiver feito, tudo dentro do diretório `/home/user/fw1-loggrabber/` poderá ser removido com o seguinte comando:  
	```
	user@debian:~/fw1-loggrabber/fw1-loggrabber$ cd
	user@debian:~$ rm -rf fw1-loggrabber/*
	```

5. Faça o download do opsec-tools [desse link](http://sourceforge.net/projects/fw1-loggrabber/files/opsec-tools/NG-FP3/). Essa ferramenta irá integrar o servidor Linux com a Manager Check Point através do canal OPSEC e a senha SIC. A ferramenta já é compilada para Linux 32bit. Copie o arquivo `opsec-tools.tar.gz` para o diretório `/home/user/` e o descomprima. Essa ferramenta poderá ser usada novamente depois, se necessário.  
	```
	user@debian:~/$ tar zxvf opsec-tools.tar.gz
	```

6. Integre o servidor Linux com a Manager Check Point via OPSEC através do comando opsec-tool. Isso gerará uma conexão TCP entre o servidor Linux e a Manager Check Point na porta TCP/18210 (FW1-ica-pull). Para rodar o comando, as seguintes informações são necessárias:
	* O endereço IP da Manager Check Point = 172.16.1.1. Isso vai na opção `-h`
	* O objecto OPSEC que foi criado = LogGrabberOPSEC. Isso vai na opção `-n`
	* A senha que foi configurada para a comunicação (SIC). Isso vai na opção `-p`  

	```
	user@debian:~$ cd opsec-tools/linux22
	user@debian:~/opsec-tools/linux22$ opsec_pull_cert -h 172.16.1.1 -n LogGrabberOPSEC -p <PASSWD>
	```  
	O resultado na shell:  
	{{< figure src="/img/09.png" >}}  

	A saída desse comando nos dá duas informações fundamentais para a integração OPSEC:

	* O arquivo opsec.p12: é o certificato usado para a comunicação entre os dois servidores
	* O Common Name (CN) do certificado, que é necessário para o fw1-loggrabber. Nesse caso, ele é `CN=LogGrabberOPSEC,O=gw..n7symj`. Agora temos tudo o que se precisa para rodas a ferramenta fw1-loggrabber.

7. Copie o arquivo `opsec.p12` para o diretório de trabalho do fw1-loggrabber:
	```
	user@debian:~/opsec-tools/linux22$ cp opsec.p12 /home/user/fw1-loggrabber
	```

8. Configure o arquivo de configuração LEA. Primeiro, copie o arquivo modelo do diretório:  
	```
	user@debian:~$ cp /usr/local/fw1-loggrabber/etc/lea.conf-sample /home/user/fw1-loggrabber/lea.conf
	```  
	Troque os parâmetros nesse arquivo lea.conf de acordo com o que adquirimos anteriormente:  
	* `lea_server ip` é o endereço IP da Manager Check Point: 172.16.1.1
	* `opsec_sic_name` é o CN do certificado que foi capturado pela ferramenta opsec-tool: `CN=LogGrabberOPSEC,O=gw..n7symj`
	* `opsec_sslca_file` é o caminho do diretório até o arquivo opsec.p12 capturado pelo opsec-tool: `/home/user/fw1-loggrabber/opsec.p12`
	* `lea_server opsec_entity_sic_name` é o Distinguished Name (DN) do certificado da Manager Check Point Manager. Essa informação pode ser coletada pelo Check Point SmartDashboard e editando o objecto da Manager do Check Point e clicando-se no botão Test SIC Status:  
	{{< figure src="/img/10.png" >}}  
	O DN pode ser encontrado no campo em evidência abaixo:  
	{{< figure src="/img/11.png" >}}  
	Nesse caso, ele é `cn=cp_mgmt,o=gw..n7symj`. Finalmente, o arquivo de configuração lea.conf tem o seguinte conteúdo:  

	```
	lea_server auth_type sslca
	lea_server ip 172.16.1.1
	lea_server auth_port 18184
	opsec_sic_name "CN=LogGrabberOPSEC,O=gw..n7symj"
	opsec_sslca_file /home/user/fw1-loggrabber/opsec.p12
	lea_server opsec_entity_sic_name "cn=cp_mgmt,o=gw..n7symj"
	```

9. É hora de se configurar o fw1-loggrabber e fornecer os parâmetros do OPSEC LEA. Para simplicidade, eu usei a configuração padrão do fw1-loggrabber disponível em `/usr/local/fw1-loggrabber/etc/`. Copie esse arquivo para o diretório de trabalho:  

	```
	user@debian:~$ cp /usr/local/fw1-loggrabber/etc/fw1-loggrabber.conf-sample /home/user/fw1-loggrabber/fw1-loggrabber.conf
	```  

	Eu troquei algumas coisas no arquivo padrão, mas a documentação desses parâmetros está bem descrita em sua página [do repositório github](https://github.com/certego/fw1-loggrabber). As seguintes linhas estão no arquivo de configuração que eu uso:  

	```
	DEBUG_LEVEL="0"
	FW1_LOGFILE="fw.log"
	FW1_OUTPUT="logs"
	FW1_TYPE="ng"
	FW1_MODE="normal"
	ONLINE_MODE="yes"
	RESOLVE_MODE="no"
	RECORD_SEPARATOR="|"
	DATEFORMAT="std"
	LOGGING_CONFIGURATION=file
	OUTPUT_FILE_PREFIX="fw1-loggrabber"
	OUTPUT_FILE_ROTATESIZE=1048576
	SYSLOG_FACILITY="LOCAL1"
	```

10. Rode a ferramenta. Note que com esse arquivo de configuração, o arquivo de saída será salvo em `fw1-lograbber.conf`. Se a ferramenta rodar corretamente, os logs serão coletados da Manager Check Point indefinidamente. Faça um favor a si mesmo e coloque o fw1-loggrabber na sua variável de ambiente PATH:

	```
	user@debian:~/fw1-loggrabber$ /usr/local/fw1-loggrabber/bin/fw1-loggrabber -c fw1-loggrabber.conf -l lea.conf
	```

## Extras
### Volume de Logs
Dependendo da sua instalação de Check Point, o volume de logs capturado pode ser gigante. Eu não gostaria de lidar com arquivos de múltiplos gigabytes de tamanho. Então eu usei o logrotate para fazer o trabalho sujo pra mim:  
```
/home/user/fw1-loggrabber/fw1-loggrabber.log{
   size 100M
   compress
   compresscmd /usr/bin/xz
   compressext .xz
   rotate 1
   copytruncate
   nocreate
   su user user
   missingok
   notifempty
}
```  
Com a diretida `lastaction` do logrotate, consegue-se fazer coisas legais com o arquivo, tipo copiando para um outro diretório de backup offline ou qualquer outra coisa, do tipo enviar para um AWS bucket.

### Parâmetros de Configuração Extra
Você vai perceber rapidamente que o arquivo de log está poluído com informações repetitivas e irrelevantes. Eu adicionei as seguintes linhas de configuração no fw1-loggrabber.conf para remover alguns campos inúteis. Note que esses campos não são documentados em lugar nenhum. Por favor, deixe-me saber se eles são.  
```
IGNORE_FIELDS=i/f_dir;i/f_name;has_accounting;uuid;product;__policy_id_tag;origin_sic_name;rule_uid;app_desc;app_id;app_category;matched_category;app_properties;app_rule_id;app_rule_name;app_sig_id;UserCheck_incident_uid
```  

Eu tentei usar a opção FW1_FILTER_RULE no arquivo de configuração, mas ela é tão limitada e cheia de problemas que eu desisti.

