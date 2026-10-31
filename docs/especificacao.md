# Especificação do OpenVNC

Versão do documento: 0.1 (planejamento), 31/10/2026.

## 1. Objetivo

Permitir que uma pessoa use um Mac a partir de um PC com Windows, vendo a tela e controlando com o mouse e o teclado do PC, sem instalar nada no Mac e sem depender de serviço em nuvem ou conta de terceiros.

O OpenVNC conecta no Compartilhamento de Tela que o macOS já traz de fábrica, sempre por dentro de um túnel SSH, e oferece um servidor MCP para que um assistente de IA (por exemplo, o Claude) possa ver a tela e agir no Mac com autorização explícita do usuário.

### Por que existe

- O Compartilhamento de Tela do macOS só aceita, por padrão, o login próprio da Apple (tipo de segurança RFB 30, "Apple Remote Desktop"). Visualizadores VNC comuns para Windows não implementam esse login, e os que implementam são fechados e exigem conta.
- Programas como AnyDesk ou TeamViewer exigem instalar um agente no Mac e conceder as permissões de Gravação de Tela e Acessibilidade na tela do próprio Mac. Isso é impossível num Mac sem mouse e teclado, ou num Mac que nunca foi configurado para acesso remoto.
- O Compartilhamento de Tela nativo não depende dessas permissões: basta ligá-lo (por exemplo, por SSH) e ele já entrega a tela.

## 2. Público

- Quem tem um Mac (mini, Studio, Pro) sem monitor, teclado ou mouse dedicados e quer usá-lo a partir do PC.
- Quem usa Mac e Windows na mesma rede e quer alternar sem trocar de mesa.
- Quem quer que um assistente de IA opere o Mac com supervisão.

## 3. Escopo

### Dentro do escopo (versão 1.0)

| Área | Requisito |
|---|---|
| Plataforma do cliente | Windows 10 22H2 e Windows 11, x64 e ARM64. |
| Plataforma do servidor | macOS com Compartilhamento de Tela e Acesso Remoto (SSH) ligados. Sem instalar nada no Mac. |
| Transporte | Túnel SSH obrigatório por padrão, usando o `ssh.exe` do Windows e autenticação por chave. |
| Autenticação de tela | RFB tipo 30 (Apple Remote Desktop). Tipo 2 (senha VNC clássica) suportado para outros servidores, desligado por padrão. |
| Imagem | Codificações Raw, CopyRect e ZRLE; pseudo-codificações DesktopSize e Cursor. |
| Entrada | Mouse completo (botões, roda, arrastar). Teclado com layout do usuário (incluindo ABNT2), Alt como Option e Windows como Command. |
| Área de transferência | Texto nos dois sentidos, nos limites do protocolo (Latin-1). |
| Interface | App nativo WPF com tema Fluent: catálogo de Macs, tela de login, sessão em janela ou tela cheia com barra flutuante. |
| Credenciais | Senha do Mac opcionalmente salva no Gerenciador de Credenciais do Windows. Nunca em arquivo próprio, log ou configuração. |
| MCP | Servidor MCP local (stdio) com ferramentas de tela, mouse, teclado e comando SSH, só ativas com a sessão aberta e autorização do usuário. |
| Idioma | Interface em português do Brasil, com estrutura preparada para inglês. |
| Distribuição | Executável único autossuficiente, publicado em GitHub Releases, com hash SHA-256. |

### Fora do escopo (1.0)

- Servidor (compartilhar a tela do Windows).
- Conexão pela internet sem VPN ou SSH exposto pelo próprio usuário. O OpenVNC não terá servidor de retransmissão.
- Áudio, transferência de arquivos por arrastar e soltar, múltiplos monitores separados.
- Clientes para macOS, Linux ou navegador.
- O modo "Alto Desempenho" da Apple, que é proprietário e só funciona entre Macs.

## 4. Requisitos funcionais

### RF1. Catálogo de Macs
- Lista de computadores salvos, cada um com nome, alias ou endereço SSH, usuário SSH, miniatura da última tela vista e estado (online, offline, verificando).
- Busca e conexão rápida por endereço digitado.
- Adicionar, editar e remover. Remover também apaga a credencial salva daquele Mac.

### RF2. Configuração do acesso SSH
- Assistente que verifica se o Mac aceita SSH por chave.
- Se não aceitar, o assistente mostra o comando exato para liberar a chave pública do PC e abre um terminal onde o próprio usuário digita a senha do Mac. O OpenVNC nunca recebe essa senha.
- Opção de usar um alias já existente no `~/.ssh/config` do usuário.

### RF3. Conexão
- Etapas visíveis: abrindo túnel SSH, conectando, autenticando, conectado.
- Erros claros e acionáveis em português: Mac desligado, SSH recusado, Compartilhamento de Tela desligado, usuário ou senha incorretos, conexão perdida.
- Reconexão automática após queda curta, mantendo a janela.

### RF4. Sessão
- Tela do Mac em janela ou tela cheia, com escala para caber na janela, tamanho real (1:1) ou ajuste inteligente.
- Barra flutuante que aparece ao levar o mouse ao topo: tela cheia, escala, qualidade, enviar atalhos (Command+Tab, Command+Espaço, forçar encerrar), área de transferência, assistente de IA (ligar e desligar) e desconectar.
- Indicadores discretos de latência e quadros por segundo.

### RF5. Teclado
- Teclas com o caractere real do layout do Windows (ABNT2, US Internacional e outros).
- Mapeamento padrão: Alt para Option, tecla Windows para Command, Ctrl para Control. Configurável.
- Captura da tecla Windows e de atalhos do sistema apenas enquanto a janela da sessão estiver em foco e em tela cheia.

### RF6. Credenciais
- Opção "Lembrar senha" grava no Gerenciador de Credenciais do Windows, com alvo `OpenVNC:<id do Mac>`.
- Sem a opção, a senha vive só em memória durante a autenticação e é apagada em seguida.

### RF7. MCP
- Executável `openvnc-mcp` que o cliente MCP (por exemplo, o Claude Code) inicia por stdio.
- Ferramentas: `listar_macs`, `estado_sessao`, `capturar_tela`, `mover_mouse`, `clicar`, `arrastar`, `rolar`, `digitar_texto`, `pressionar_teclas`, `executar_comando`.
- Só funcionam com uma sessão aberta no app e com o interruptor "Permitir assistente" ligado naquela sessão. `executar_comando` (SSH) exige um segundo interruptor próprio.
- Toda ação do assistente aparece na barra da sessão, em tempo real, e fica registrada no histórico local da sessão.

## 5. Requisitos não funcionais

| Tema | Meta |
|---|---|
| Latência | Até 80 ms entre o movimento do mouse e a resposta na tela, em rede local cabeada. Até 150 ms em Wi-Fi. |
| Fluidez | 30 quadros por segundo em uso de escritório a 1080p em rede local. |
| Consumo | Abaixo de 10% de CPU num processador de 4 núcleos com a tela parada; abaixo de 250 MB de memória. |
| Inicialização | Catálogo visível em até 1,5 s. |
| Segurança | Seção 7 e `SECURITY.md`. |
| Acessibilidade | Navegação completa por teclado no app, contraste AA, leitores de tela no catálogo e nas caixas de diálogo. |
| Dependências | Apenas componentes do .NET, pacotes mantidos pela Microsoft e o SDK oficial do MCP para C#, com versões fixadas. |

## 6. Abordagem

Detalhada em [arquitetura.md](arquitetura.md). Resumo: solução .NET 10 com quatro projetos (núcleo do protocolo, app WPF, servidor MCP e testes). O núcleo implementa o protocolo RFB 3.8 com o login da Apple, sem dependências de interface. O app desenha a tela num `WriteableBitmap` e conversa com o servidor MCP por um named pipe restrito ao usuário do Windows.

## 7. Segurança

- Túnel SSH obrigatório por padrão. A conexão direta na porta 5900 existe só como opção avançada, com aviso explícito, porque o fluxo de imagem do Compartilhamento de Tela não é criptografado depois do login.
- A chave do túnel é a do próprio usuário (OpenSSH do Windows). O OpenVNC não guarda chave privada nem senha de SSH.
- Senha do Mac: só em memória ou no Gerenciador de Credenciais do Windows. Nunca em log, arquivo de configuração, título de janela ou mensagem de erro.
- O MCP só aceita conexões do mesmo usuário do Windows (ACL do named pipe) e só age com autorização explícita na sessão. O assistente nunca recebe a senha do Mac.
- Nenhuma telemetria. Nenhuma conexão de rede além do túnel SSH escolhido pelo usuário.

## 8. Riscos

| Risco | Impacto | Mitigação |
|---|---|---|
| A Apple mudar o login tipo 30 numa versão futura do macOS. | Conexão para de funcionar. | Testes de integração com o macOS mais recente a cada versão; suporte opcional ao tipo 2. |
| Falhas de segurança no Compartilhamento de Tela da Apple (alerta SANS de agosto de 2026). | Exposição do Mac na rede local. | Túnel SSH obrigatório; documentação para limitar a porta 5900; recomendação de desligar quando não usar. |
| Desempenho do RFB pior que o RDP em vídeo e animação. | Experiência abaixo da expectativa. | ZRLE, cursor local, pedidos incrementais e medição contínua; expectativa clara no README. |
| Mapeamento de teclado incompleto em layouts menos comuns. | Teclas erradas. | Tabela de testes por layout; mapeamento configurável. |
| Uso indevido do MCP por outro programa local. | Ações não autorizadas no Mac. | ACL por usuário, interruptor por sessão, registro de ações e aviso visual. |
| Marca "VNC" registrada pela RealVNC e projetos com nome parecido. | Pedido de troca de nome. | Aviso no README de que o projeto não tem relação com a RealVNC; nome do pacote isolado para facilitar troca. |

## 9. Critérios de pronto (1.0)

1. Conectar num Mac com macOS atual, sem nada instalado nele, por túnel SSH, e controlar com mouse e teclado ABNT2 sem erros de caractere.
2. Metas de latência e fluidez da seção 5 medidas e registradas no repositório.
3. Testes unitários do protocolo, do login tipo 30 e dos decodificadores passando no CI.
4. Roteiro de teste de ponta a ponta executado num Mac real e documentado.
5. MCP testado no Claude Code: capturar tela, clicar, digitar e executar comando, todos bloqueados sem autorização e liberados com ela.
6. Release com executável único para x64 e ARM64, com hash SHA-256 publicado.
7. README, `SECURITY.md` e guia de primeiro uso completos em português.

## 10. Marcos

| Marco | Entrega |
|---|---|
| M0 | Planejamento: especificação, arquitetura, licença e repositório. |
| M1 | Núcleo do protocolo: handshake, login tipo 30, Raw e CopyRect, testes unitários e teste manual contra Mac real por linha de comando. |
| M2 | Sessão mínima em WPF: ver e controlar, túnel SSH automático. |
| M3 | ZRLE, cursor local, área de transferência, teclado completo e reconexão. |
| M4 | Catálogo, assistente de SSH, credenciais e acabamento visual Fluent. |
| M5 | Servidor MCP com autorização por sessão. |
| M6 | Empacotamento, CI, release 1.0 e documentação de uso. |
