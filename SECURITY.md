# Política de segurança

O OpenVNC dá acesso à tela e ao teclado de um computador. Falhas de segurança aqui são sérias, e agradecemos quem as relata com responsabilidade.

## Como relatar

Use o recurso **Report a vulnerability** na aba **Security** deste repositório no GitHub (aviso privado de segurança). Não abra issue pública para vulnerabilidades.

Inclua, se possível:

- versão do OpenVNC, do Windows e do macOS;
- passos para reproduzir;
- impacto esperado;
- sugestão de correção, se tiver.

Respondemos o recebimento em até 7 dias e combinamos com você o prazo de divulgação.

## Escopo

- Código deste repositório: protocolo, login da Apple, túnel SSH, app, servidor MCP e empacotamento.
- Fora do escopo: falhas no macOS, no Compartilhamento de Tela da Apple ou no OpenSSH. Relate essas falhas aos respectivos fornecedores. Se uma delas afetar usuários do OpenVNC, avise também, para documentarmos a mitigação.

## Recomendações de uso

- Mantenha o túnel SSH ligado (padrão). Não exponha a porta 5900 do Mac na rede nem na internet.
- Use autenticação por chave no SSH.
- Desligue o Compartilhamento de Tela no Mac quando não estiver usando.
- Só ligue "Permitir assistente" e "Permitir comandos" quando estiver acompanhando a sessão.
