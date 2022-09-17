---
title: "PlantUML and Latex"
date: 2022-09-10T08:57:34+05:45
tags: ["hugo", "plantuml", "latex"]
author: "thapabishwa"
summary: "Exploring rich content capabilites of hugo"
katex: math
markup: 'mmark'
draft: true
---

## Plantuml Diagram
{{<diagram name="component-diagram" type="plantuml">}}
@startuml

skinparam component {
    FontColor          black
    AttributeFontColor black
    FontSize           17
    AttributeFontSize  15
    AttributeFontname  Droid Sans Mono
    BackgroundColor    #6A9EFF
    BorderColor        black
    ArrowColor         #222266
}

title "OSCIED Charms Relations (Simple)"
skinparam componentStyle uml2

cloud {
    interface "JuJu" as juju
    interface "API" as api
    interface "Storage" as storage
    interface "Transform" as transform
    interface "Publisher" as publisher
    interface "Website" as website

    juju - [JuJu]

    website - [WebUI]
    [WebUI] .up.> juju
    [WebUI] .down.> storage
    [WebUI] .right.> api

    api - [Orchestra]
    transform - [Orchestra]
    publisher - [Orchestra]
    [Orchestra] .up.> juju
    [Orchestra] .down.> storage

    [Transform] .up.> juju
    [Transform] .down.> storage
    [Transform] ..> transform

    [Publisher] .up.> juju
    [Publisher] .down.> storage
    [Publisher] ..> publisher

    storage - [Storage]
    [Storage] .up.> juju
}

@enduml
{{</diagram>}}

## Adding LATEX

$$\[ x^n + y^n = z^n \]$$
$$\int_{a}^{b} x^2 dx$$
$$S (ω)=1.466\, H_s^2 \,  \frac{ω_0^5}{ω^6 }  \, e^[-3^ { ω/(ω_0  )]^2}$$
