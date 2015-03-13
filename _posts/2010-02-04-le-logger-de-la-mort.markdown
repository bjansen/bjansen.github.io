---
layout: post
title:  "Le logger de la mort"
date:   2010-02-04 16:24:00
categories: Java
---
<p>Parfois, au cours d'une mission chez un client, on tombe sur du code pas testé, et pas toujours prévu pour être testé non plus. Je vais présenter un cas sur lequel je suis tombé récemment, issu d'un assez gros projet Java EE en cours de développement.</p>

Une classe de services X effectue une certain nombre de traitements. En cas d'erreur, au lieu de jeter une exception métier, un code d'erreur est loggué. Impossible d'utiliser un <code>@Test(expected=...)</code>, donc. Fort heureusement, le champ logger dans X est une interface, potentiellement mockable :

{% highlight java %}
interface Ilogger {
  public abstract void logError(final Key key, final Object... messages);
  // blabla
}
{% endhighlight %}

Dans tous les cas où nous sommes intéressés, la méthode logError est appelée avec un paramètre Key représentant le code d'erreur. Malheureusement, le logger n'est pas injecté dans un setter, et il est privé. On aimerait tout de même vérifier que l'erreur est bien logguée sous certaines conditions. Solution, un peu de réflexion pour forcer le champ logger à une instance que l'on contrôle :

{% highlight java %}
public class MockLogger implements ILogger {
  private List<Key> loggedKeys;

  public MockLogger() {
    super();
    loggedKeys = new ArrayList&lt;Key&gt;();
  }

  public void logError(Key key, Object... messages) {
    loggedKeys.add(key);
  }

  public boolean wasLogged(Key key) {
    return loggedKeys.contains(key);
  }

  public void resetLogCalls() {
    loggedKeys.clear();
  }
}
{% endhighlight %}

La méthode <code>resetLogCalls</code> permet d'utiliser la même instance du logger sur plusieurs tests, il faut l'appeler dans une méthode annotée <code>@After</code>.

Il faut maintenant injecter notre propre logger dans le service testé :

{% highlight java %}
Field f = serviceUnderTest.getClass().getDeclaredField("logger");
f.setAccessible(true);
f.set(serviceUnderTest, myLoggerInstance);
f.setAccessible(false);
{% endhighlight %}

Et voila, nous pouvons maintenant faire des <code>assertTrue</code> utilisant la méthode <code>wasLogged</code> et la Key nous intéressant ! Notez l'appel à <code>Field#setAccessible(boolean)</code> pour contourner le caractère private du champ <code>logger</code> (que l'on restaure par la suite).

Note : si le logger n'avait pas utilisé d'interface, nous aurions pu nous en sortir en créant une classe fille redéfinissant toutes les méthodes sans appels à <code>super()</code>.