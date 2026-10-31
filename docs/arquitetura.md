# Arquitetura do OpenVNC

Versão do documento: 0.1 (planejamento), 31/10/2026. Requisitos em [especificacao.md](especificacao.md).

## 1. Visão geral

```
 PC com Windows                                              Mac
┌──────────────────────────────────────────────┐          ┌───────────────────────────┐
│ OpenVNC.App (WPF)                             │          │                           │
│  ├─ Catálogo, login, sessão                   │          │  sshd (Acesso Remoto)     │
│  ├─ OpenVNC.Core ── RfbClient ─┐              │          │     │                     │
│  │                             │ 127.0.0.1:N  │  SSH     │     ▼ 127.0.0.1:5900      │
│  └─ SshTunnel (ssh.exe -L) ────┴──────────────┼─────────►│  screensharingd (RFB)     │
│         ▲                                     │ porta 22 │                           │
│         │ named pipe (só o usuário do Windows)│          └───────────────────────────┘
│ openvnc-mcp (stdio) ◄── cliente MCP (Claude)  │
└──────────────────────────────────────────────┘
```

- No Mac não se instala nada. São usados dois serviços nativos: o Acesso Remoto (SSH) e o Compartilhamento de Tela (RFB na porta 5900).
- Todo o tráfego de tela passa por um túnel SSH aberto com o `ssh.exe` do próprio Windows.
- O app é o único dono da sessão. O servidor MCP é um processo separado e fino, que pede ações ao app por um named pipe.

## 2. Decisões

| Decisão | Escolha | Motivo | Alternativas descartadas |
|---|---|---|---|
| Linguagem e runtime | C# no .NET 10 (LTS) | Nativo no Windows, criptografia e `BigInteger` embutidos, SDK oficial do MCP, publicação em executável único. | Rust (mais código de janela, desenho e entrada feito à mão); .NET Framework 4.x (sem suporte a recursos atuais). |
| Interface | WPF com tema Fluent do .NET | Visual do Windows 11 sem biblioteca externa, `WriteableBitmap` composto pela GPU, maduro e estável. | WinUI 3 (distribuição em executável único mais difícil); React em WebView2 (cópia de cada quadro para o navegador, latência). |
| Transporte | `ssh.exe` do Windows com `-L` | Criptografia e autenticação por chave já resolvidas pelo OpenSSH, sem biblioteca SSH de terceiros, reaproveita `~/.ssh/config`. | Biblioteca SSH em .NET (dependência de terceiro); conexão direta (fluxo de imagem sem criptografia). |
| Login da Apple | Implementação própria do tipo 30 | Única forma de usar o Compartilhamento de Tela sem software extra. Referência pública: noVNC e LibVNCServer. | Tipo 2 com senha VNC (exige configurar no Mac, mais fraco, alvo de falhas conhecidas). |
| Ícones | Segoe Fluent Icons | Já vêm no Windows 11, mesmo conjunto do sistema. | Pacotes de ícones externos. |
| Testes | MSTest | Mantido pela Microsoft, alinhado à política de dependências. | xUnit, NUnit. |
| Comunicação app e MCP | Named pipe com ACL por SID | Local, sem porta de rede, restrito ao usuário do Windows. | HTTP em localhost (qualquer processo local acessa). |
| Credenciais | Gerenciador de Credenciais do Windows | Protegido pelo login do Windows, gerenciável pelo usuário no Painel de Controle. | Arquivo próprio cifrado com DPAPI. |

## 3. Estrutura do repositório

```
OpenVNC/
├─ OpenVNC.slnx
├─ Directory.Build.props          versão, nullable, avisos como erro, analisadores
├─ Directory.Packages.props       versões centralizadas e fixadas
├─ global.json                    versão do SDK .NET fixada
├─ src/
│  ├─ OpenVNC.Core/               biblioteca, sem dependência de interface
│  │  ├─ Rfb/
│  │  │  ├─ RfbClient.cs          ciclo de vida da sessão e laço de mensagens
│  │  │  ├─ RfbStream.cs          leitura e escrita big-endian com buffer
│  │  │  ├─ Handshake.cs          versão e negociação do tipo de segurança
│  │  │  ├─ Messages/             mensagens cliente e servidor
│  │  │  └─ Encodings/            Raw, CopyRect, ZRLE, DesktopSize, Cursor
│  │  ├─ Security/
│  │  │  ├─ ArdAuthenticator.cs   tipo 30: Diffie-Hellman, MD5, AES-128-ECB
│  │  │  └─ VncAuthenticator.cs   tipo 2: DES (opcional)
│  │  ├─ Framebuffer/             buffer BGRA, retângulos sujos, cursor
│  │  ├─ Input/                   keysyms X11, mapeamento de teclas e mouse
│  │  ├─ Transport/
│  │  │  └─ SshTunnel.cs          processo ssh.exe, porta livre, saúde do túnel
│  │  └─ Hosts/                   catálogo de Macs (JSON) e credenciais
│  ├─ OpenVNC.App/                WPF
│  │  ├─ Views/                   Catálogo, Login, Sessão, Assistente SSH
│  │  ├─ ViewModels/
│  │  ├─ Rendering/               ponte do framebuffer para WriteableBitmap
│  │  ├─ Interop/                 gancho de teclado, DWM (Mica), Credential Manager
│  │  ├─ Automation/              servidor do named pipe para o MCP
│  │  └─ Resources/               estilos Fluent, textos pt-BR e en
│  └─ OpenVNC.Mcp/                executável openvnc-mcp (stdio)
│     ├─ Program.cs
│     ├─ Tools/                   ferramentas MCP
│     └─ AppPipeClient.cs
├─ tests/
│  ├─ OpenVNC.Core.Tests/         protocolo, login, decodificadores, teclado
│  └─ OpenVNC.Mcp.Tests/          contrato das ferramentas e autorização
├─ docs/
│  ├─ especificacao.md
│  ├─ arquitetura.md
│  └─ teste-ponta-a-ponta.md      roteiro manual com Mac real (a partir do M1)
├─ .github/workflows/ci.yml       build e testes em windows-latest (a partir do M1)
├─ README.md
├─ SECURITY.md
└─ LICENSE.md
```

Os diretórios de código são criados no marco em que ganham conteúdo, sem pastas vazias antes disso.

## 4. Núcleo do protocolo (OpenVNC.Core)

### 4.1 Sequência de conexão

1. `SshTunnel` escolhe uma porta livre em `127.0.0.1`, inicia `ssh -N -o BatchMode=yes -o ExitOnForwardFailure=yes -L 127.0.0.1:<porta>:127.0.0.1:5900 <alias>` sem janela e espera a porta aceitar conexão. A saída de erro do `ssh` vira mensagem de erro em português.
2. Versão: o servidor da Apple anuncia `RFB 003.889`. O cliente responde `RFB 003.008`.
3. Segurança: o servidor lista os tipos aceitos (o Mac testado oferece 30, 33, 35 e 36). O cliente escolhe 30.
4. Login tipo 30:
   - Lê `gerador` (u16), `tamanhoChave` (u16), `primo` e `chavePublicaServidor` (ambos com `tamanhoChave` bytes, big-endian).
   - Gera chave privada aleatória com `RandomNumberGenerator`, calcula a pública e o segredo compartilhado com `BigInteger.ModPow`.
   - Chave AES = MD5(segredo com `tamanhoChave` bytes).
   - Bloco de 128 bytes preenchido com aleatórios; usuário UTF-8 terminado em zero nos primeiros 64, senha UTF-8 terminada em zero nos 64 seguintes.
   - Cifra com AES-128-ECB sem preenchimento, envia os 128 bytes e a chave pública do cliente.
   - Lê `SecurityResult`; em falha, lê o motivo e devolve erro tipado.
   - Os buffers com a senha são zerados com `CryptographicOperations.ZeroMemory` logo após o uso.
5. `ClientInit` compartilhado, leitura do `ServerInit`, `SetPixelFormat` (32 bits, BGRA, little-endian) e `SetEncodings` (ZRLE, CopyRect, Raw, Cursor, DesktopSize).
6. Laço de leitura numa thread dedicada; pedidos `FramebufferUpdateRequest` incrementais a cada atualização concluída.

### 4.2 Decodificação e framebuffer

- O framebuffer é um `int[]` BGRA de largura × altura, com uma lista de retângulos sujos protegida por lock curto.
- ZRLE usa um único fluxo zlib por conexão (`ZLibStream` do .NET), com blocos de 64 × 64 e paletas, conforme a RFC 6143.
- A pseudo-codificação Cursor desenha o cursor localmente no app, o que reduz a latência percebida do mouse.
- DesktopSize recria o framebuffer quando a resolução do Mac muda.

### 4.3 Entrada

- Mouse: posição convertida da janela escalada para o framebuffer; botões 1, 2 e 3; roda como botões 4 e 5 (e 6 e 7 na horizontal).
- Teclado: teclas especiais por tabela de códigos virtuais do Windows para keysyms X11. Caracteres pelo resultado do layout ativo (`ToUnicodeEx`), com keysym Latin-1 ou `0x01000000 | ponto de código`.
- Tecla Windows: gancho `WH_KEYBOARD_LL` instalado só enquanto a sessão está em foco e em tela cheia, removido ao perder o foco.

## 5. App (OpenVNC.App)

- Padrão MVVM com implementação própria mínima (sem pacote de MVVM externo).
- Renderização: o framebuffer é copiado para um `WriteableBitmap` Bgr32 no evento `CompositionTarget.Rendering`, só nos retângulos sujos (`AddDirtyRect`), uma vez por quadro de tela.
- Escala com `BitmapScalingMode` de alta qualidade, ou vizinho mais próximo no modo 1:1.
- Tema Fluent nativo do WPF, segue claro ou escuro do Windows, fundo Mica via DWM no Windows 11.
- Textos em arquivos de recurso, com pt-BR como padrão.
- Catálogo de Macs em `%APPDATA%\OpenVNC\hosts.json` (sem segredos). Miniaturas em `%LOCALAPPDATA%\OpenVNC\thumbs\`.

## 6. MCP (OpenVNC.Mcp)

### 6.1 Por que um processo separado

O cliente MCP inicia o servidor por stdio e controla o ciclo de vida dele. O app precisa continuar aberto e visível, com a sessão sob controle do usuário. Por isso o `openvnc-mcp` não conecta no Mac: ele só repassa pedidos ao app aberto.

### 6.2 Canal com o app

- Named pipe `\\.\pipe\OpenVNC.Automation.<SID do usuário>`, com ACL que só permite o SID do usuário atual.
- Mensagens JSON delimitadas por tamanho, no formato pedido e resposta com `id`, `ferramenta`, `argumentos`, `resultado` ou `erro`.
- Sem sessão aberta ou sem autorização, o app responde erro tipado e o MCP repassa ao assistente uma mensagem clara.

### 6.3 Ferramentas

| Ferramenta | Faz | Autorização |
|---|---|---|
| `listar_macs` | Lista Macs do catálogo e qual está em sessão. | Sempre. |
| `estado_sessao` | Resolução, conectado ou não, autorização ativa. | Sempre. |
| `capturar_tela` | PNG da tela atual, com escala opcional. Vem do framebuffer, sem permissão extra no Mac. | "Permitir assistente". |
| `mover_mouse`, `clicar`, `arrastar`, `rolar` | Ações de mouse em coordenadas da tela do Mac. | "Permitir assistente". |
| `digitar_texto`, `pressionar_teclas` | Texto e combinações (ex.: `cmd+space`). | "Permitir assistente". |
| `executar_comando` | Comando no Mac pelo SSH, com tempo limite e saída truncada. | "Permitir comandos", separado. |

Cada ação aparece na barra da sessão e entra no histórico local da sessão.

## 7. Segurança

Ameaças consideradas e respostas da arquitetura:

| Ameaça | Resposta |
|---|---|
| Alguém na rede local lê a tela. | Túnel SSH obrigatório por padrão. |
| Senha do Mac vaza do PC. | Só em memória ou no Gerenciador de Credenciais; buffers zerados; nunca em log. |
| Outro programa local controla o Mac pelo MCP. | Pipe com ACL por SID, autorização por sessão, aviso visual e histórico. |
| Assistente executa comando destrutivo. | Interruptor separado para comandos, tempo limite e registro de cada comando. |
| Servidor malicioso envia dados inválidos. | Validação de tamanhos e limites em todas as leituras; resolução máxima; erro e desconexão em mensagem desconhecida. |
| Atualização falsa do app. | Releases só no GitHub, com hash SHA-256; assinatura de código planejada para depois da 1.0. |

## 8. Testes

- Unitários (MSTest): leitura e escrita do protocolo, login tipo 30 com vetores fixos (primo, chaves e textos cifrados gerados por implementação independente), ZRLE com amostras capturadas, mapeamento de teclado por layout.
- Fuzzing leve dos decodificadores com entradas aleatórias, sem travar e sem estourar memória.
- Integração manual com Mac real seguindo `docs/teste-ponta-a-ponta.md` a cada versão.
- CI no GitHub Actions: build, testes e analisadores em `windows-latest`.

## 9. Distribuição

- `dotnet publish` autossuficiente, executável único, `win-x64` e `win-arm64`.
- Release no GitHub com notas, hash SHA-256 e o executável do MCP junto.
- Configuração do MCP no Claude Code documentada no README.
