+++
date = "2016-03-23T00:00:00+01:00"
title = "Hardening de SSH para sua VPS"
Description = "Configuração segura de SSH para setups simples"

translationurl = "/post/ssh-hardening/"

tags = [
    "SSH",
    "Hardening"
]
+++

## TL;DR
VPS - Virtual Private Servers - normalmente vem com um monte de configurações padrão. Nesse post eu descrevo como configurar o servidor SSH de forma segura para um servidor simples, como um VPS onde eu sou o único usuário e eu tenho controle total do servidor e do cliente.

## Intro
Um dos benefícios de se ter uma VPS 
One of the things of having a VPS - Virtual Private Server - é a liberdade de fazer o que quiser com um sistema que está no ar o tempo todo [(ou quase o tempo todo)](https://www.ovh.com/us/dedicated-cloud/discover/sla.xml). Eu uso OVH por razões que me fogem da memória. Na verdade, isso não importa. Onde quer que o servidor esteja, seja na Amazon AWS, Microsoft Azure, Linode, um Raspberry Pi rodando na sua sala ou o seu [provedor "underground"/privado favorito que aceita bitcoins](http://libertyvps.net/), ele não contará com uma configuração padrão ao nível de um administrator consciente de segurança.

Na atualidade, o SSH é o de facto standard para acesso remoto. Os tempos áureos do telnet em um hub de 10Mbps, onde conseguia-se sniffar tudo, estão acabados. Apesar disso, configurar e usar criptografia é um desafio, e SSH não é diferente.

O objetivo principal de usar SSH e um túnel criptografado entre dois computadores é proteger o conteúdo que trafega entre eles através do canal encriptado - e também muitos outros atributos de segurança, como autenticação e integridade de mensagem. Se esse túnel for comprometido por algum motivo, más coisas podem acontecer.

Existem [vários](http://www.coresecurity.com/content/ssh-insertion-attack) [tipos](https://blog.sucuri.net/2013/07/ssh-brute-force-the-10-year-old-attack-that-still-persists.html) [de ataques](https://sites.google.com/site/clickdeathsquad/Home/cds-ssh-mitmdowngrade) que focam no lado criptográfico do SSH e não no protocolo propriamente dito. Dá um Google em "SSH attacks" pra você ver.

No decorrer desse post eu vou mostrar as fontes de informação que eu coletei para criar a configuração do meu SSH. Eu não vou fazer desenhos de como SSH funciona, sobre key exchange, MACs ou qualquer outra coisa porque já existem muitos sites por aí com explicações brilhantes. Eu também não entrarei em muitos detalhes em relação a criptografia propriamente dita- apesar de ser um tópico muito interessante, não é o foco desse post.

Vale aqui a menção ao [blog post do stribika](https://stribika.github.io/2015/01/04/secure-secure-shell.html) que me inspirou a fazer tudo isso, apesar de não estar fazendo isso para me proteger de uma certa agência norte-americana de três letras que não deve ser nomeada. Segurança está livremente disponível e eu usar a melhor que tem. Ponto.

## O Início
Primeiro de tudo, é necessário entender algo sobre SSH que eu relevei por MUITO tempo. Conectar um computador com o outro envolve DOIS computadores. *Não é suficiente* otimizar a configuração somente do servidor SSH. Veja referências ao [CVE-0216-0777 e CVE-0216-0778](https://www.digitalocean.com/community/tutorials/how-to-fix-openssh-s-client-bug-cve-0216-0777-and-cve-0216-0778-by-disabling-useroaming) em relação ao UseRoaming e também sobre esse excelente trabalho de [Filippo Valsorda](https://github.com/FiloSottile/whosthere) onde ele identifica quem você é pela chave SSH que você usa.

Eu vou começar com a configuração do servidor e depois vou para o cliente. *NÃO PULE A CONFIGURAÇÃO DO CLIENTE* como eu fiz por muitos anos da minha vida.

## O Servidor

Você recebe o servidor VPS. Beleza! Você sabe quais são as chaves SSH que o servidor está usando e quais opções de criptografia que foram usados para criá-las? Muitos desses VPS se valem de configurações automáticas de geração de chave do pacote do servidor openssh durante seu provisionamento automático, o que é mais ou menos tudo bem. O fato é que eu não confio o suficiente no provedor (para criação da chave SSH, e não por ter a habilidade de acessar todos os meus dados armazenados em disco e em memória, e roubar todas as minhas chaves).


Existem *histórias de horror* como [esse aqui](http://wiki.hetzner.de/index.php/Ed25519/en) onde Hetzner, um provedor de VPS, reusou a mesma chave do servidor de SSH em uma série de clientes devido a um erro de configuração.

`Due to an error in the installation software introduced on April 10th, 2015, the Ed25519 SSH host keys (/etc/ssh/ssh_host_ed25519_key) on our standard images were no longer automatically regenerated. This resulted in identical Ed25519 SSH host keys for each affected OS image.` -- Hetzner

Lembrando. Eu sou o único usuário desse sistema. Eu decido qual é o nível de criptografia eu quero e eu não me importo com "backward compatibility" e clientes SSH velhos. Meu objetivo é usar o maior nível de criptografia possível, evitar ataques de "downgrade", dificultar a descriptografia do meu tráfego por atacantes, etc. Eu crio e protejo minhas próprias senhas.

1. Recrie as chaves do seu servidor SSH  
	Não existe razão para o servidor oferecer múltiplas chaves para autenticação do servidor. A ferramenta `ssh-keygen` oferece uma série de opções para o tipo de chave para criar. Minha opção é sempre usar [ed25519](https://ed25519.cr.yp.to/) mesmo que RSA com uma chave suficientemente grande seja igualmente seguro. Minha chave então se chama `/etc/ssh/ssh_server_ed25519`.  
	
	```
	ssh-keygen -t ed25519 -f /etc/ssh/ssh_server_ed25519 -o -a 128
	```

	Essa chave não pode ter um passphrase. É assim que SSH funciona. Você reinicia o servidor, o SSH inicializa e tudo funciona. Tenha certeza de que as permissões para o novo arquivo de chave criado é 600 (-rw-------).  
	Note o parâmetro `-o -a 128`. Esse é o número de rodadas usado para derivar a chave. O padrão é 16. Nossos computadores poderosos são capazes de fazer essa matemática em 2 segundos, mais ou menos. Existe uma boa explicação para isso [aqui](https://martin.kleppmann.com/2013/05/24/improving-security-of-ssh-private-keys.html)  
	Para evitar falha de configuração, certifique-se de remove todas as chaves antigas que estavam no servidor anteriormente.  

	Também, note a digital (fingerprint) das chave pública que acabou de ser gerada. A saída é parecida com essa:

	```
	The key fingerprint is:
	ed:9a:3d:e6:dd:61:4d:99:cd:68:09:6c:b0:af:e2:4e example.com
	```

	Você pode verificar isso depois com o seguinte comando:  
	```
	ssh-keygen -lf /etc/ssh/ssh_server_ed25519.pub
	```

2. Reconfigure o arquivo `/etc/ssh/sshd_config`  
	A maioria das configurações e explicações estão aqui. Abaixo está a cópia do meu próprio `sshd_config` e vamos linha por linha. Algumas linhas já são a configuração padrão do SSH, mas eu quero ter certeza total do comportamento que eu quero.
	```
	Port 22
	Protocol 2
	HostKey /etc/ssh/ssh_server_ed25519
	AllowGroups ssh-allowed
	ClientAliveInterval 300
	ClientAliveCountMax 0
	KexAlgorithms curve25519-sha256@libssh.org
	Ciphers chacha20-poly1305@openssh.com
	MACs hmac-sha2-512-etm@openssh.com
	UsePrivilegeSeparation yes
	SyslogFacility AUTH
	LogLevel VERBOSE
	LoginGraceTime 120
	PermitRootLogin no
	StrictModes yes
	PubkeyAuthentication yes
	IgnoreRhosts yes
	HostbasedAuthentication no
	PermitEmptyPasswords no
	ChallengeResponseAuthentication no
	PasswordAuthentication no
	X11Forwarding no
	PrintMotd no
	PrintLastLog yes
	TCPKeepAlive yes
	UseLogin no
	PermitTunnel no
	AcceptEnv LANG LC_*
	Subsystem sftp /usr/lib/openssh/sftp-server
	UsePAM no
	```

	Aqui está a explicação para os pedaços importantes:

	2.1) `Port 22`  
		Somente aceite conexões na porta 22 em todas as interfaces de rede do servidor. Mudar isso para um outro valor por motivos de "segurança" significa segurança por obscuridade e é ridículo, na minha opinião. Ao invés disso, é melhor usar um firewall local que filtre conexões de entrada baseados no endereço IP de origem.

	2.2) `Protocol 2`  
		Porque protocol 1 já está ultrapassado e não oferece mais segurança.

	2.3) `HostKey /etc/ssh/ssh_server_ed25519`  
		A nova chave do servidor que se acabou de criar.

	2.4) `AllowGroups ssh-allowed`  
		Somente usuários do grupo Linux `ssh-allowed` são permitidos de entrar através de SSH. Isso é pra proteger a mim mesmo de possíveis erros, tipo criando um usuário temporário e acabar afetando a segurança do servidor.

	2.5) `ClientAliveInterval 300` and `ClientAliveCountMax 0`  
		Tempo de inatividade do usuário em segundos. Isso é forçado no lado do servidor.

	2.6) `KexAlgorithms curve25519-sha256@libssh.org`  
		Não existe razão para usar múltiplos algoritmos de key exchange. Usar ECDH com Curve25519 com SHA2 é perfeitamente ok e eu não quero ligar pra moduli com tamanhos de bit insuficientes no cado de Diffie-Hellman group exchanges.

	2.7) `Ciphers chacha20-poly1305@openssh.com`  
		Essa é a criptografia simétrica que finalmente criptografa o conteúdo indo pelo túnel entre o cliente e servidor. Esse cipher oferece o melhor disponível, evita análise de tráfego por um atacante, e por aí vai.

	2.8) `MACs hmac-sha2-512-etm@openssh.com`  
		Para a integridade da comunicação, esse algoritmo fornece Encrypt-then-MAC com a maior tamanho de chave disponível.

	2.9) `UsePrivilegeSeparation yes`  
		Não rode o daemon ssh com uma conta privilegiada. Essa configuração cria processos filhos não-privilegiados para lidar com o tráfego de rede. Se o servidor for comprometido através do daemon ssh, o atacante receberá acesso com poucos privilégios e precisará trabalhar mais para escalar privilégios.

	2.10) `LogLevel VERBOSE`  
		Temos muito espaço em disco, então porque não fazer log de todas as informações disponíveis de um dos serviços mais importantes do meu VPS.

	2.11) `PermitRootLogin no`  
		Essa é a configuração default, mas alguns VPS te dão acesso remoto direto com root por default. Você não quer usar root pra acesso remoto ao servidor. Crie seu próprio usuário não privilegiado para se conectar remotamente e somente escale privilégios quando necessário.

	2.12) `StrictModes yes`  
		Isso me protege contra falha de configuração. Ele checa se as permissões dos arquivos de chave, entre outras coisas, estão devidamente configuradas.

	2.13) `PubkeyAuthentication yes`  
		Permita autenticação com chaves públicas. Isso é fundamental para o que estamos querendo fazer e desabilitar autenticação por senha.

	2.14) `IgnoreRhosts yes` and `HostbasedAuthentication no`  
		Esse é o padrão, mas eu quero ter certeza.

	2.15) `PasswordAuthentication no` and `PermitEmptyPasswords no`  
		Eu quero fazer login usando uma chave, e não com a senha do meu usuário no Linux.

	2.16) `ChallengeResponseAuthentication no`  
		Esse é complicado e eu uso as duas formas. Até o manual do sshd-config é difícil de entender. Se eu quero somente me autenticar usando minha chave, eu coloco `no`. Mais a frente eu discuto duplo-fator de autenticação, onde será necessário mudar essa configuração pra `yes`.

	2.17) `X11Forwarding no`  
		Eu não uso X11, o padrão é `no` mas eu quero ter certeza.

	2.18) `UseLogin no`  
		Esse é o padrão, mas eu quero ter certeza de que `login(1)` não será usado para autenticação.
	
	2.19) `PermitTunnel no`  
		Esse é o padrão, mas eu quero ter certeza de que eu somente uso SSH para conectar a VPS, nada mais.

	2.20) `UsePAM no`  
		Essa configuração é parecida com o ChallengeResponseAuthentication. Se eu estou usando somente a chave para autenticação, eu coloco `no`. Se eu uso qualquer outra coisa, eu uso `yes`.

## O Cliente
A configuração do cliente é muito semelhante a do servidor. Você cria chaves fortes, define algoritmos critográficos para tudo, disabilita coisas inúteis, pronto.

Mas o ponto mais importante aqui é o uso de chaves diferentes para servidores diferentes. No passado, eu usava apenas uma chave e adicionava ela no arquivo authorized_keys de todos os servidores em que eu tinha acesso. No final, eu não tinha mais idéia de onde as chaves estavam, quais servidores eu tinha acesso, etc. O risco aqui é que se sua chave privada for comprometida, perdida, roubada, copiada sem querer no Dropbox/Github/USB, vai ser muito difícil saber o tamanho do estrago.

Por isso, eu proponho o uso de uma chave diferente para cada servidor eu gerencio. Sim, isso cria um stress gerencial, mas estamos falando sobre melhores hábitos de segurança e, no final das contas, eu não tenho acesso a muitos servidores mesmo. Eu estou falando sobre meu próprio VPS. Então porque não criar uma chave separada para acessá-lo? Isso também é válido para serviços como Github e tudo o que usa SSH.

O arquivo de configuração, do ponto de vista do usuário (e não do root), é `~/.ssh/config`. Pelo amor de `StrictModes`, tudo dentro do meu diretório `.ssh` tem permissão `0600`. Os passos descritos abaixo estão todos contidos nesse arquivo.

1. Configure o comportamento padrão:
	Essa parte precisa seguir o que configuramos no servidor SSH no VPS. Eu quero dizer ao meu cliente SSH para somente usar aqueles algoritmos especificos de criptogrfia, com aquele método de key exchange com aquele cipher específico. A entrada do `Host *` fornece configuração padrão global para todos os outros hosts, então eu não preciso ficar repetindo os mesmos parâmetros em cada entrada `Host`:

	```
	Host *
	    KexAlgorithms curve25519-sha256@libssh.org
	    Ciphers chacha20-poly1305@openssh.com
	    MACs hmac-sha2-512-etm@openssh.com
	    ForwardAgent no
	    UseRoaming no
	```

	No exemplo acima, note as duas linhas de configuração extras:  

	1.1) `ForwardAgent no`
		Essa é a configuração padrão, mas eu quero ter certeza de que se eu começar a usar o ssh-agent para gerenciar minhas chaves privadas, eu não quero que ele faça "forward" delas para servidores remotos.

	1.2) `UseRoaming no`  
		Esse é a funcionalidade miserável e não documentada que poderia expor minhas chaves privadas a atacantes no caso de um ataque MITM.

	Isso será aplicado a todos os servidores que eu conectar daqui em diantes. Se um servidor necessitar de um parâmetro diferente, crie uma entrada específica para aquele Host. Você vai perceber rapidamente que essa configuração vai quebrar o acesso a alguns servidores, *incluindo Github*. Mais adiante eu mostro o exemplo de configuração específica para o Github.

2. Crie uma chave para cada servidor que você tem acesso:  
	Nesse exemplo, eu criei uma específica para o Github. Não se esqueça de usar um passphrase complexo para criptografar sua chave:  
	```
	ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_github -C "user@computer" -o -a 128
	``

	O comentário aqui com a opção `-C` é importante para que você consiga identificar facilmente de onde aquela chave pertence. Imagine em um servidor, onde você conecta com 10 máquinas diferentes. Qual chave pública pertence a qual máquina? Então, faça isso com esse comentário.

3. Configure `~/.ssh/config` para servidores específicos:

	```
	Host github.com
	    Hostname ssh.github.com
	    Port 443
	    MACs hmac-sha2-512
	    IdentityFile ~/.ssh/id_ed25519_github
	```

	Aqui está a [ajuda do Github](https://help.github.com/articles/using-ssh-over-the-https-port/) em como configurar. Note que eu precisei adicionar o MAC específico que o Github aceita. Infelizmente, 

	Para o seu próprio VPS, o processo é similar. Crie uma nova chave, crie o blocl `Host` no seu arquivo `~/.ssh/config`, adicione sua chave pública no arquivo `authorized_keys` no usuário do servidor, e pronto. Aqui está a entrada para o meu próprio VPS:  

	```
	Host example.com
		ServerAliveInterval 30
		StrictHostKeyChecking yes
		IdentityFile ~/.ssh/id_ed25519_example.com
	```

	Nesse exemplo, eu adiciono a opção `StrictHostKeyChecking` para ter certeza de que se alguma coisa acontecer com a chave do meu servidor - caso o "fingerprint" mudou, ou estou sofrendo um ataque de MITM, qualquer coisa - eu sou bloqueado de conectar. Isso previne a troca automática do fingerprint do servidor no meu arquivo `known_hosts`, que seria catrastrófico.

4. Adicione sua chave no ssh-agent:
	Sério, você não quer continuar digitando pra sempre a sua super senha a cada vez que você se conecta no servidor VPS. Usando o ssh-agent faz com que seja mais difícil para atacantes se eles tiverem comprometido seu sistema local e tudo o que você digita é capturado por eles. Então, imagine isso. Mesmo que o atacante tenha uma cópia da sua chave privada, veja tudo o que você digita, mas não seja capaz de entrar na sua VPS de outro computador que não o seu. Certamente ele poderia adicionar uma chave que ele mesmo controla no arquivo authorized_keys do servidor, mas isso já levantaria umas suspeitas. É aqui que o duplo fator de autenticação entra em jogo.

## Duplo Fator de Autenticação

Na minha opinião, autenticar com uma chave que eu controlo e protejo já é o suficiente. Mesmo que minha chave privada seja comprometida, ela está criptografada com um passphrase forte e difícil de se quebrar com brute-force. Meu servidor SSH não permite nada a não ser a minha chave de autenticação, então as credenciais do meu usuário local no servidor Linux está seguro. Eu também posso bloquear os endereços IP de origem dos atacantes se tentarem brute-force com ferramentas como [fail2ban](http://www.fail2ban.org/wiki/index.php/Main_Page).

Mas existem alguns casos em que você gostaria de usar um outro fator de autenticação. Isso pode ser alcançado com o `google-authenticator`, onde você usa o seu celular para gerar um OTP - One Time Passwords - que é necessário para autenticação, além da sua chave e passphrase. Então mesmo que sua chave e seu passphrase sejam comprometidos pelo atacante, ele não consegue se autenticar porque ele não tem o seu celular.

Em um próximo post eu discuto em mais detalhes o ssh-agent e o duplo fator de autenticação.
