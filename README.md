# OpenVNC

Use o seu Mac a partir do PC com Windows, com mouse e teclado do PC, sem instalar nada no Mac e sem conta em serviço de terceiros.

O OpenVNC conecta no Compartilhamento de Tela que o macOS já traz de fábrica, sempre por dentro de um túnel SSH, e oferece um servidor MCP para que um assistente de IA possa ver e operar o Mac com a sua autorização.

> Estado: **planejamento (M0)**. Ainda não há versão para baixar. Acompanhe os marcos em [docs/especificacao.md](docs/especificacao.md#10-marcos).

## Por que

- O Compartilhamento de Tela do macOS exige o login próprio da Apple. Os visualizadores VNC comuns para Windows não fazem esse login, e os que fazem são fechados e pedem conta.
- Ferramentas como AnyDesk precisam de um agente no Mac e de permissões que só podem ser dadas na tela do próprio Mac, o que é impossível num Mac sem mouse e teclado.
- O Compartilhamento de Tela nativo não precisa dessas permissões. Basta ligá-lo.

## Como vai funcionar

1. No Mac, ligue **Acesso Remoto** e **Compartilhamento de Tela** em Ajustes do Sistema > Geral > Compartilhamento. Pelo Terminal, o Acesso Remoto liga com `sudo systemsetup -setremotelogin on` ou, se o macOS exigir Acesso Total ao Disco para isso, com `sudo launchctl load -w /System/Library/LaunchDaemons/ssh.plist`.
2. No PC, o OpenVNC configura o acesso SSH por chave com você e abre o túnel sozinho.
3. Você entra com o usuário e a senha do Mac e usa o Mac numa janela ou em tela cheia.

## Destaques planejados

- App nativo para Windows 10 e 11 (WPF, tema Fluent, claro e escuro).
- Túnel SSH obrigatório por padrão, usando o OpenSSH que já vem no Windows.
- Login da Apple (RFB tipo 30) implementado do zero, sem bibliotecas de terceiros.
- Teclado com o seu layout (ABNT2 incluído), Alt como Option e Windows como Command.
- Senha opcionalmente guardada no Gerenciador de Credenciais do Windows.
- Servidor MCP com autorização por sessão: o assistente só vê e age quando você permite, e tudo o que ele faz aparece na tela.
- Sem telemetria.

## Documentação

- [Especificação](docs/especificacao.md): objetivo, escopo, requisitos, riscos e critérios de pronto.
- [Arquitetura](docs/arquitetura.md): componentes, protocolo, MCP, segurança e estrutura do código.
- [Segurança](SECURITY.md): como relatar vulnerabilidades.

## Licença

[PolyForm Noncommercial 1.0.0](LICENSE.md). Você pode usar, estudar, modificar e compartilhar para fins não comerciais, incluindo uso pessoal, educacional, por instituições de caridade e órgãos públicos. Uso comercial não é permitido.

Por isso o OpenVNC é **código disponível**, não "open source" no sentido da Open Source Initiative, que não admite restrição de uso comercial.

## Aviso de marca

VNC é marca registrada da RealVNC Ltd. O OpenVNC é um projeto independente, sem relação com a RealVNC ou com a Apple. macOS e Mac são marcas da Apple Inc.

---

**English:** OpenVNC is a native Windows client for the built-in macOS Screen Sharing (Apple RFB security type 30), tunneled over SSH, with an MCP server for supervised AI control. Source-available under PolyForm Noncommercial 1.0.0. Planning stage.
