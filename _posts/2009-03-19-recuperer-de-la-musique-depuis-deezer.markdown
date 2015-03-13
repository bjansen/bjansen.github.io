---
layout: post
title:  "Récupérer de la musique depuis Deezer"
date:   2009-03-19 14:15:00
categories: scripts
---
<p>Même si je ne suis pas trop fan de <a href="http://www.deezer.com">Deezer</a> pour diverses raisons (interface Flash trop lourde, beaucoup trop de pub, ...), je suis quand même bien content de pouvoir y rechercher une musique que j'ai en tête et que je peux récupérer pour la réécouter par la suite...</p>

<p>En effet, il est de notoriété publique que ce système met en cache la chason en cours d'écoute et la stocke dans un fichier flv nommé /tmp/Flash* sous linux.</p>
<p>À partir de là, il est facile de créer un script shell qui va se charger de récupérer ce ficher, puis d'en extraire la musique en MP3 et de la sauvegarder dans notre dossier de musique favori. Top du top, le script que je vous propose par la suite permet également d'éditer les tags ID3 et de renommer le fichier en conséquence, puis ajoute le morceau à la playlist Amarok en cours :D</p>
<p>Voici la bête :</p>

{% highlight bash %}
#!/bin/bash

DEST=/home/user/Musique

cd $DEST

files=`ls /tmp/Flash* | cut -d '/' -f3`

for file in $files
do
        ffmpeg -i "/tmp/$file" -acodec copy -f mp3 "$DEST/$file.mp3"
        id3ren -quick -notrack -template "%a - %s.mp3" "$DEST/$file.mp3"
        dcop amarok playlist addMedia "$DEST/$file"
done
{% endhighlight %}