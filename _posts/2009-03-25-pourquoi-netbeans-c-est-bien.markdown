---
layout: post
title:  "Pourquoi Netbeans c'est bien"
date:   2009-03-25 15:16:00
categories: Netbeans
---

<p>C'est un fait avéré, Netbeans c'est mieux qu'Eclipse. <small>(comment ça, un troll tout poilu ?)</small>. Pas plus tard qu'il y a une minute, j'en ai refait l'heureuse expérience. Agacé de voir une série de getters/setters dans ma classe, qui sont triviaux à comprendre mais qui prennent de la place, j'ai voulu utiliser l'<a href="http://wiki.netbeans.org/FaqCustomCodeFolds">editor folding</a>, propre à Netbeans (je crois, du moins). Ça consiste juste en deux commentaires autour du code que l'on veut <i>folder</i> (euh... comment traduit-on ça en français, déjà...) :

{% highlight java %}
// <editor-fold defaultstate="collapsed" desc="getters and setters">
public Foo getFoo() { }
public void setFoo(Foo foo) { }
// </editor-fold>
{% endhighlight %}

Maintenant vous aurez un joli '-' sur la gauche qui permettra de cacher tout ça et qui affichera la description voulue (ici, "getters and setters").</p>

<p>Amusons-nous encore un peu, en transformant ça en <a href="http://wiki.netbeans.org/Java_EditorUsersGuide#section-Java_EditorUsersGuide-HowToUseLiveTemplates">code template</a> :</p>
<img src="/images/netbeans/code-templates.png" alt="Code templates dans Netbeans" />

<p>Et c'est parti, sélectionnez un bloc de texte que vous ne voulez plus voir, puis cliquer sur l'ampoule dans la gutter et choisissez "Surround with an editor fold tag". Magique, non ?</p>