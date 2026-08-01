+++
title = 'Pequenas automações'
date = 2026-07-31T23:45:53-03:00
draft = false
+++

Ha um pouco de tempo eu mudei de emulador de terminal para o Kitty. Porém uma das coisas que senti muita falta nele foi o tmux-sessionizer, que eu usava frequentemente.

Ele é uma ferramenta simples que com um atalho de teclado ou com um alias lista os projetos dentro de diretórios relevantes e os mostra em uma lista para selecionar e abri-los.

A verdade é que provavelmente sem a facilidade de navegação que ele traz eu provavelmente nunca teria começado a usar o terminal tanto quanto uso atualmente. Porque precisar ficar digitando comandos o tempo todo para entrar em um projeto pode ser um pouco chato, e é exatamente o tipo de atrito pequeno que te desanima de abrir uma ferramenta que no final pode ser muito melhor.

Ha um tempo atrás eu só abria o terminal ao lado de um projeto enquanto desenvolvia, agora, eu comecei a usar para quase tudo. E pequenas ferramentas como essa foram importantes para isso.

Após a migração, eu precisava muito de algo para suprir essa necessidade, e consegui resolver isso com essa simples função que coloquei no meu bashrc:

```bash
prj() {
    SELECTED=$({ find "$HOME/source" -maxdepth 1; find "$HOME/projects" -maxdepth 1; find "$HOME/.config" -maxdepth 1; } | fzf \
        --layout=reverse \
        --margin=5 \
        --highlight-line  \
        --style=full \
        --border=bold --color="bg+:#444444,pointer:#af5fff")

    if [ -n "$SELECTED" ]; then
        cd "$SELECTED"
        NAME="${PWD##*/}"
        if type kitty &> /dev/null;
        then
            kitten @ set-tab-title $NAME
        fi;
    fi;
}
```

Como pode ver, nem tudo isso era necessário, uma boa parte das linhas dele foram só para estilização visual mesmo.

E uma coisa que eu amo é como essas pequenas automações conseguem afetar positivamente a nossa experiência como desenvolvedores. Acredito que esse foi o principal motivo pelo qual eu comecei a usar tanto o terminal, porque é muito mais fácil fazer isso nele do que em ferramentas bem polidas que usam UI visual.
