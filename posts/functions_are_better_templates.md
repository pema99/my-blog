---
slug: functions-are-better-templates
title: Breaking the Twig; Using functions as a DSL for HTML Templating
description: Imagine yourself (or myself actually) wanting to put your thoughts on the internet. Groundbreaking I know. And since you are a fellow Software Engineer, you obviously go to make yourself a website instead of doing the sensible thing and using a generator. Do you get out your favourite backend MVC framework with a PWA Webscale Vue TypeScript Frontend? Or maybe you grab yourself a Microframework and just start writing Pug Templates. Either way, I believe we can do it simpler and easier to maintain!
date: 2019-12-13
---


Imagine yourself (or myself actually) wanting to put your thoughts on the internet. Groundbreaking I know. And since you are a fellow Software Engineer, you obviously go to make yourself a website instead of doing the sensible thing and using a generator. Do you get out your favourite backend MVC framework with a PWA Webscale Vue TypeScript Frontend? Or maybe you grab yourself a Microframework and just start writing Pug Templates.
Either way, I believe we can do it simpler and easier to maintain!  

## Ze Problem

Humans have tried for *millenia* (measured in software engineerung units) to solve the problem of taking a url and spitting out web pages. Today I want to show you an approach in just a few lines of F# to create a simple DSL for generating HTML which is extensible, modular and concise through the power of functional programming!  
  
We are going to replicate the features people expect from a regular templating language and the core of that magic is just 5 lines of code!   
  
  
But what would we expect from a templating language in 2019 and how are we going to archieve that with just functions? Well let's look at what a popular templating language like [Twig](https://twig.symfony.com/doc/3.x/templates.html) advertises as it's features: 

* __IDE Integration:__ Our templating language is just F#, so we are good!
* __(Named-) Variables:__ Function parameters!
* __Filters:__ More functions!
* __Control Structures:__ All our usual goodies, including *match*
* __Comments:__ Yes
* __Including other templates:__ Even more functions!
* __Template inheritance:__ Functions with parameters!
* __Escaping:__ This one is kinda tricky, obviously we can escape using *functions*, but forcing the user to do so isn't trivial
* __Macros:__ More *drumroll* functions
* __Math/Logic Expressions:__ Duh  
  
But before we look at how to archieve those things let's just get some html tags going.

## An HTML Primer - I promise it's short
If you don't know what HTML looks like you probably live under a rock next to a pineapple under the sea, but here is a primer anyway:  

HTML is organized in tags, enclosed in brackety bois `<>`, which represent a hierarchical structure, XML gives greetings.
Each element is composed of a tag and a closing tag, the only difference being: closing tags begin with a slash. Tags can also have attributes which are key-value pairs with the value usually written in quotation marks. And tags can't also just contain plain text which is usually what we see on our web pages!  
  
So let's model everyone's favourite town in HTML!
```xml
<bikini-bottom>
    <crusty-crab>
        <mr-crabs color="red"></mr-crabs>
        <a href="formeloni.html" hidden>It's probably just mustard</a>
    </crusty-crab>
    <white-bread></white-bread>
</bikini-bottom>
```  
  
Easy enough, so let's get started on trying to generate some of this.

## Hello HTML
To start off let us print some HTML to the console!  
This will let us nicely test our DSL and if you want to you could also pipe the output to a file and open it in a web browser. But without further hesitation, let's get nostalgic with everyone's favourite program:  
  
```fs
"<p>Hello World</p>"
    |> printfn "%A"
```  
  

Alright we are done, time to pack up.  
Or it would be, but then we'd just serve static HTML files on an apache toaster.  
  
So let's do the obligatory thing and make our website greet us with our name!

```fs
sprintf "<p>Hello %s</p>" "Sally"
    |> printfn "%A"
```  
  

Pretty good! You could probably use this to write a website if you wanted to and had an inclination towards masochism, but let's try and start working on making ourselves an actually useful api for generating these tags.  
  
First of all we want to be able to give a tag name and have it spit out the according HTML tag. So `tag p` becomes `<p></p>` and `tag div` becomes `<div></div>`. We are also going to want to be able to pass in some text and have it be placed between our opening and closing tags. The function to do both of these things I am from here on out gonna call `element` and it is the backbone of the entire DSL (that magic 10 line function I mentioned in the beginning).  
  
For now we can implement it as such: 
```fs
let element
    (tag: string)
    (inner: string) =

    sprintf "<%s>%s</%s>" tag inner tag
```  
  

and we can already use it to implement our previous example:
```fs
element "p" "Hello Sally"
    |> printfn "%A"
```  
  

And actually this is already composable in a way!  
  
```fs
element "div" (
              element "p" "Hello Sally!" +
              element "p" "How are you?"
              )
    |> printfn "%A"
```  
  

The `+` operator sprinkled in there is kind of annoying though, so what if we were to build that functionality directly into the `element` function? And while we are at it we are also going to make some helper functions, using partial application to make the code a bit prettier:  
  
```fs
let element
    (tag: string)
    (children: string list) =

    sprintf "<%s>%s</%s>" tag (children |> String.concat "") tag

let div   = element "div"
let p     = element "p"


div [
  p ["Hello Sally"]
  p ["How are you?"]
] |> printfn "%A"
```  
  
So far so good. This is pretty simple to write and already very versatile. We just stack more of these and we get to build ourselves some nice HTML templates.  
We still have one problem though:  
  
Some tags have "attributes". A hyperlink (or *anchor tag*) for example needs to know where it points to when someone wants to visit that reference like so: `<a href="https://outside.nature">The door</a>`.  
  
So let's extend our element function to support that!  
I opted to take attributes as tuple of `string * string`: keys and values. There is other ways to do it, but this one is fairly straightforward.  
  
```fs
let element 
    (tag: string) 
    (attrs: (string * string) list) 
    (children: string list): string = 

    let attrsString = 
        attrs
            |> Seq.map (fun (n, v) -> sprintf " %s=\"%s\"" n v)
            |> String.concat ""
  
    sprintf "<%s%s>%s</%s>" tag (attrsString) (children |> String.concat "") tag
```  
  
Okay so building the attribute string is a bit messy, we turn every pair of strings into a string with the syntax `key="value"` and then concatenate them, making sure to leave one space before every attribute.  But don't worry we don't have to look at this function ever again! This is the 5 line magic function I was speaking of in the beginning and that is all there is too it! All other features can be built on top of this skeleton!  

So let's use it:
  
```fs
let a = element "a"

a [("href", "https://outside.nature")] [
    "The door"
] |> printfn "%A"
```  
  
  
Alright that is only slightly ugly and because this way of doing it is tedious, error prone (imagine typo-ing "href", yikes) and we are functional programmers, we again use helper functions to make this a bit pretter:  
  
```fs
let attribute (name: string) (value: string) = name, value
let href = attribute "href"

a [href "https://outside.nature"] ["The door"]
    |> printfn "%A"
```
  
And we are done! This is how you can go and build your website from now on!  
Or maybe not quite, but maybe I'll make a follow-up post where I show how I used this to make this blog! But as for the DSL part of this, we are done. The rest is just functions.  
  
  
If you are interested in this style of server-side templating, check out the [repo](https://github.com/Nyrox/my-blog) for this website. The DSL is contained entirely in one File and contains all the predefined tag names, attributes and helpers I use:  
[https://github.com/Nyrox/my-blog/blob/master/backend/src/Html.fs](https://github.com/Nyrox/my-blog/blob/master/backend/src/Html.fs)

