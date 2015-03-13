---
layout: post
title:  "Un switch annoté en Java"
date:   2010-07-02 12:05:00
categories: Java
---
Un billet sur le blog de <a href="http://agilitateur.azeau.com/post/2010/07/01/Un-switch-%C3%A9volutif-en-C">l'Agilitateur</a> répondant à un billet sur le blog <a href="http://blog.excilys.com/2010/06/25/de-switcher-nest-pas-jouer">d'Excilys</a> concernant les switchs en java me donne envie de parler un peu de codage en Java. Cet article est une variante du code C# proposé par Oaz en Java.

<p>Je vais vous épargner toute la discussion sur le switch c'est bien / c'est nul, il y a déjà les <a href="http://blog.excilys.com/2010/06/25/de-switcher-nest-pas-jouer/comment-page-1/">commentaires enflammés</a> sur le blog d'Excilys pour ça.</p>
<p>
Passons directement au code : de la même manière qu'Oaz l'a fait, je dispose d'une interface Function qui définit le contrat que vont suivre les différents cas du switch. Pour que l'on sache que les différentes sous-classes sont bien des cases, on les annote avec un @Case créé pour l'occasion :

{% highlight java %}
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Case {
    String name();
}
{%endhighlight %}

On demande au compilateur de garder cette annotation au runtime (pour faire du scanning dynamique), et on précise qu'elle peut s'appliquer à un type (classe ou interface).
</p>

<p>On peut alors utiliser l'annotation de cette façon :

{% highlight java %}
public interface Function {
    void doYourJob();
}

@Case(name="bar")
public class BarFunction implements Function {

    public void doYourJob() {
        System.out.println("On va manger... des chips ! T'entends ? DES CHIPS !!");
    }
}

@Case(name="foo")
public class FooFunction implements Function {

    public void doYourJob() {
        System.out.println("Hi, my name is foo!");
    }
}

@Case(name="othercase")
public class OtherCase {

    public void doYourJob() {
        System.out.println("I'm not gonna be called :-(");
    }
}
{%endhighlight %}
</p>

<p>Reste maintenant le plus dur à faire : scanner un package à la recherche de classes annotées @Case et qui héritent de notre classe de base (ici Function, mais on va essayer de rester génériques).

{% highlight java %}
public class Switch<T> {

    private String packageName;
    private Map<String, T> cases;
    private Class<T> parentType;
    
    public Switch(String packageName, Class<T> parentType) {
        this.packageName = packageName;
        this.parentType = parentType;
        scanCases();
    }

    private void scanCases() {
        cases = new HashMap<String, T>();

        try {
            for (Class clazz : getClasses(packageName)) {
                if (clazz.isAnnotationPresent(Case.class) && isA(clazz, parentType)) {
                    Case theCase = (Case) clazz.getAnnotation(Case.class);
                    cases.put(theCase.name(), (T) clazz.newInstance());
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

        System.out.println("Scanned " + cases.size() + " cases.");
    }

    private boolean isA(Class clazz, Class<T> class1) {
        if (clazz.getSuperclass() == class1) {
            return true;
        }
        for (Class intf : clazz.getInterfaces()) {
            if (intf == class1) {
                return true;
            }
        }
        return false;
    }

    public T on(String caseName) throws IllegalArgumentException {
        if (cases.containsKey(caseName)) {
            return cases.get(caseName);
        } else {
            throw new IllegalArgumentException("The case " + caseName + " does not exist");
        }
    }

    private List<Class> getClasses(String pckgname) { ... }
{%endhighlight %}

J'ai fait appel au <a href="http://java.sun.com/j2se/1.5.0/docs/guide/language/generics.html">generic type</a> introduit en Java5, cela permet d'avoir un switch acceptant n'importe quelle classe de base qu'implémentent les différents cas. J'ai été obligé de repasser cette information dans le constructeur car en Java il n'est pas possible de faire un T.class dans la phase de scan des classes (pour savoir si la classe scannée étend ou implémente T). À la place, on a donc un Class<T> en second paramètre.</p>
<p>La méthode scanCases va parcourir toutes les classes d'un package (grâce à la méthode getClasses() honteusement <a href="http://forums.sun.com/thread.jspa?messageID=3946549#3946549">trouvée sur le forum de <strike>Sun</strike> Oracle</a>). Pour chaque classe, on regarde si elle est annotée @Case et si elle étend ou implémente T. Si c'est le cas, on en ajoute une instance dans une Map avec le nom du cas comme clé.</p>
<p>Dernière étape : la méthode on() qui renvoie l'instance associée à un cas ou jette une exception s'il n'existe pas.</p>
<p>La glue entre tout ça, c'est un main assez simple : 

{% highlight java %}
public class Main {

    public static void main(String[] args) {
        Switch<Function> swi = new Switch<Function>("com.jansen.annotatedswitch.demo", Function.class);
        
        swi.on("foo").doYourJob();
        swi.on("bar").doYourJob();
        swi.on("othercase").doYourJob();
    }

}
{%endhighlight %}

Notez le dernier appel qui va planter : j'ai bien une classe annotée @Case(name="othercase") mais elle n'implémente pas Function (une ruse de sioux pour égarer le lecteur peu attentif ;-) ).</p>

<p>Conclusion : c'est aussi possible en Java, avec un peu de gymnastique intellectuelle pour arriver à faire ce qu'on souhaite avec les generics :P</p>