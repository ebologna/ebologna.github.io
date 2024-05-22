---
title: Pirámides de Población con ggplot2
author: Eduardo Bologna
summary: Una forma fácil de representar distribuciones de población por sexo y edad
date: '2020-04-10'
slug: piramides-2
categories: []
tags: ["pirámides de poblacion", "ggplot2", "demografía"]
bibliography: library.bib
---

Las pirámides ofrecen una visión rápida de la estructura por sexos y edades de una población. Representan las edades en el eje vertical, los volúmenes o proporciones de población en el eje horizontal y en las barras horizontales se grafican hacia la derecha el total de mujeres de cada edad y a la izquierda el de varones.
Veremos cómo generar estas pirámides de población por medio del paquete `ggplot2`, por lo que se supone un conocimiento general del uso del paquete.
Se verá el modo de construir pirámides con datos provenientes de una base de microdatos o de una tabla de clasificación por sexos y edades. Para los ejemplos, se usan datos provenientes de la Encuesta Permanente de Hogares (EPH) de Argentina aplicada en el tercer trimestre de 2018. El diccionario de variables y categorías está disponible en https://www.indec.gob.ar/ftp/cuadros/menusuperior/eph/EPH_diseno_reg_t414.pdf.
En primer lugar se lee la base de microdatos (disponible  en https://www.indec.gob.ar/ftp/cuadros/menusuperior/eph/EPH_usu_3_Trim_2018_txt.zip ):
```{r}
eph.3.18 <- read.table("usu_individual_T318.txt",
  sep = ";", header = TRUE)
```


Y se cargan dos paquetes necesarios:
```{r warning=FALSE}
library(ggplot2)
library(ggthemes)
```

## A partir de microdatos

### En valores absolutos
Si se dispone de la base de microdatos, la pirámide parte de un gráfico de barras, que cuenta la cantidad de repeticiones de cada edad. Por el modo en que se elabora la base de la EPH, es necesaria una corrección previa en la variable edad (codificada como CH06).

```{r}
eph.3.18$edad<-eph.3.18$CH06
eph.3.18$edad[eph.3.18$edad==-1]<-0
```

Además, se redefine sexo y se etiqueta:

```{r}
eph.3.18$sexo<-as.factor(eph.3.18$CH04)
levels(eph.3.18$sexo)<-c("varon", "mujer")
```

El recurso consiste en construir dos gráficos de barras, con la variable edad tomada de cada uno de dos subconjuntos de datos: el que contiene solo mujeres y el que tiene solo varones. Por eso los datos de origen se sitúan en el `geom_bar` y no en `ggplot`. En el de varones, se multiplica por -1 la cuenta de casos. Luego se rotan los ejes y se indican los rótulos del eje horizontal.
```{r}
ggplot()+ geom_bar(data=subset(eph.3.18, eph.3.18$sexo=="mujer"),
                   aes(edad,fill=sexo))+
  geom_bar(data=subset(eph.3.18, eph.3.18$sexo=="varon"),
                   aes(edad, fill=sexo, y=..count..*(-1)))+
  coord_flip()+
  scale_y_continuous(breaks=seq(-500,500,100),labels=abs(seq(-500,500,100)))+
  ylab("Volumen")+xlab("Edades simples")
```

![jpg](./volumen_edades_simples_df.jpg)


### En relativos

La lógica de construcción es la misma, pero debe indicarse que, en lugar de mostrar la *cuenta* de casos, se tome la *proporción*, a la que también se multiplica por -1 para los varones. En la sintaxis anterior no fue necesario indicar `y=..count..` para las mujeres, porque es la operación por defecto de `geom_bar`, ahora que pedimos `y=..prop..`, hay que indicarlo en los dos `geom_bar`. La expresión `group=1` es necesaria según lo explica @Wickham2009.
El comando `paste0` le *pega* el signo *%* al resultado de la operación (la secuencia en este caso). Si se una `paste` en lugar de `paste0`, queda un espacio entre el número y el signo *%*.
```{r}
ggplot()+ geom_bar(data=subset(eph.3.18, eph.3.18$sexo=="mujer"),
                   aes(edad,fill=sexo, y=..prop.., group=1))+
  geom_bar(data=subset(eph.3.18, eph.3.18$sexo=="varon"),
           aes(edad, fill=sexo, y=(..prop..)*(-1), group=1))+
  coord_flip()+
  scale_y_continuous(breaks=seq(-.02,.02,.01),
                     labels=paste0(100*abs(seq(-.02,.02,.01)),"%"))
```
![jpg](./relativos_edades_simples_df.jpg)


## A partir de la distribución

### En absolutos

Se construye la tabla y se la trata como `data.frame`, se la etiqueta y se calcula el total de casos, para usarlo luego. A la columna *casos*, donde sexo sea varón se cambia de signo. La diferencia con lo anterior en el argumento de `geom_bar` es que hay que pedir `stat="identity"`, para que en lugar de contar ocurrencias de la categoría, tome a la columna *casos* como frecuencias.
```{r}
sexo_edad<-table(eph.3.18$edad, eph.3.18$sexo)
sexo_edad<-data.frame(sexo_edad)
names(sexo_edad)<-c("edad", "sexo", "casos")
n<-sum(sexo_edad$casos)

sexo_edad$casos[sexo_edad$sexo=="varon"]<--sexo_edad$casos

ggplot(sexo_edad)+ geom_bar(aes(edad, casos,
fill=sexo), stat = "identity")+ coord_flip()+
  scale_y_continuous(breaks=seq(-500,500,100),labels=abs(seq(-500,500,100)))

```

![jpg](./volumen_edades_simples_tabla.jpg)

### En relativos

Ahora el eje debe representar frecuencias relativas, por lo que se agrega columna que se construye como los *casos* dividido el total de observaciones (n).
```{r}
sexo_edad$relativas<-sexo_edad$casos/n

ggplot(sexo_edad)+ geom_bar(aes(edad, relativas,
fill=sexo), stat = "identity")+ coord_flip()+
  scale_y_continuous(breaks=seq(-.01,.01,.005),
                     labels=paste0(100*abs(seq(-.01,.01,.005)),"%"))

```

![jpg](./relativos_edades_simples_tabla.jpg)


## Pirámide con edades agrupadas

### Definición de las categorías de edad

En la función `cut` se indican los puntos de corte y sus etiquetas, definidos primero como dos vectores. Para la categoría abierta final, elegimos un punto extremo de corte en 150 años. Por defecto los intervalos incluyen al límite superior, para evitarlo, y respetar la convención en demografía, se pide `right = FALSE`.
```{r}

cortes_edad <- c(0,5,10,15,20,25,30,35,40,45,50,55,60,65,70,75,80,85,150)
clases_edad <- c("0-4","5-9","10-14","15-19","20-24","25-29","30-34",
               "35-39","40-44","45-49","50-54","55-59","60-64","65-69",
               "70-74","75-79","80-84","85+")

eph.3.18$edad_qq<-cut(eph.3.18$edad,breaks = cortes_edad,
                                right = FALSE,
                                labels = clases_edad)

```

Para hacer la pirámide con edades agrupadas desde la base de microdatos, se procede igual que antes, con la nueva variable `edad_qq`. Luego se agregan detalles de estilo.
```{r}
ggplot()+ geom_bar(data=subset(eph.3.18, eph.3.18$sexo=="mujer"),
                   aes(edad_qq,fill=sexo))+
  geom_bar(data=subset(eph.3.18, eph.3.18$sexo=="varon"),
                   aes(edad_qq, fill=sexo, y=..count..*(-1)))+
  coord_flip()+
  scale_y_continuous(breaks=seq(-2000,2000,500),labels=abs(seq(-2000,2000,500)))+
  xlab("Edades quinquenales")+ylab("Volumen")+
  scale_fill_manual(values = c("varon"="green", "mujer"="orange"))+
  theme_tufte()+labs(title="Distribución por sexo y edad de la muestra de
  la Encuesta Permanente de Hogares",
  subtitle="Aglomerados Urbanos de Argentina - Tercer Trimestre 2018",
  caption="Fuente: INDEC 2020")
```

![jpg](./agrupardas.jpg)

## Algunas pirámides de grupos específicos

### Población estudiantil universitaria

Se define el grupo de estudiantes en la universidad como el compuesto por quienes tienen nivel de educación "universitario incompleto" (NIVEL_ED=5) y están asistiendo a un establecimiento educativo (CH10=1).  Se establece un límite de edad en 60 años, para evitar 10 observaciones extremas que interfieren en la visualización del conjunto.
```{r}
estudiantes.3.18<-subset(eph.3.18, eph.3.18$NIVEL_ED==5 & eph.3.18$CH10==1 &
                           eph.3.18$edad<61)

```

Y se construye la pirámide para esa nueva matriz de datos

```{r}
ggplot()+ geom_bar(data=subset(estudiantes.3.18, estudiantes.3.18$sexo=="mujer"),
                   aes(edad,fill=sexo))+
  geom_bar(data=subset(estudiantes.3.18, estudiantes.3.18$sexo=="varon"),
                   aes(edad, fill=sexo, y=..count..*(-1)))+
  coord_flip()+
  scale_y_continuous(breaks=seq(-500,500,100),labels=abs(seq(-500,500,100)))+
  scale_x_continuous(breaks=seq(0,75, 15))+
  xlab("Edades quinquenales")+ylab("Volumen")+
  scale_fill_manual(values = c("varon"="green", "mujer"="orange"))+
  theme_tufte()+labs(title="Distribución por sexo y edad de estudiantes de nivel universitario
  en la muestra de la EPH",
  caption="Aglomerados Urbanos de Argentina - Tercer Trimestre 2018 \n Fuente: INDEC 2020")
```

![jpg](./universitaries.jpg)

### En condición de desocupación

El grupo de personas desocupadas es el de quienes cumplen con ESTADO = 2:
```{r}
desocupades.3.18<-subset(eph.3.18, eph.3.18$ESTADO==2)
```

Y se construye la pirámide para esa nueva matriz de datos


```{r}
ggplot()+ geom_bar(data=subset(desocupades.3.18, desocupades.3.18$sexo=="mujer"),
                   aes(edad,fill=sexo))+
  geom_bar(data=subset(desocupades.3.18, desocupades.3.18$sexo=="varon"),
                   aes(edad, fill=sexo, y=..count..*(-1)))+
  coord_flip()+
  scale_y_continuous(breaks=seq(-500,500,100),labels=abs(seq(-500,500,100)))+
  scale_x_continuous(breaks=seq(0,75, 15))+
  xlab("Edades quinquenales")+ylab("Volumen")+
  scale_fill_manual(values = c("varon"="green", "mujer"="orange"))+
  theme_tufte()+
  labs(title="Distribución por sexo y edad de personas desocupadas en la muestra
  de la EPH",
  caption="Aglomerados Urbanos de Argentina - Tercer Trimestre 2018 \n Fuente: INDEC 2020")
```

![jpg](./desocupades.jpg)

### Jefas y jefes de hogar

Esta categoría es la que corresponde a CH03=1
```{r}
jefxs.3.18<-subset(eph.3.18, eph.3.18$CH03==1)
```


```{r}
ggplot()+ geom_bar(data=subset(jefxs.3.18, jefxs.3.18$sexo=="mujer"),
                   aes(edad,fill=sexo))+
  geom_bar(data=subset(jefxs.3.18, jefxs.3.18$sexo=="varon"),
                   aes(edad, fill=sexo, y=..count..*(-1)))+
  coord_flip()+
  scale_y_continuous(breaks=seq(-500,500,100),labels=abs(seq(-500,500,100)))+
  scale_x_continuous(breaks=seq(0,75, 15))+
  xlab("Edades quinquenales")+
  ylab("Volumen")+
  scale_fill_manual(values = c("varon"="green", "mujer"="orange"))+
  theme_tufte()+
  labs(title="Distribución por sexo y edad de jefas y jefes de hogar en la muestra
  de la EPH",
  caption="Aglomerados Urbanos de Argentina - Tercer Trimestre 2018 \n Fuente: INDEC 2020")
```

![jpg](./jefxs.jpg)
