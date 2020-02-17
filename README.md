# Estadisitca-IMC
 Analaisis estadístico sobre el IMC

# Introducción

En este proyecto se evaluaron las bases de datos de la Encuesta Nacional de Salud y Nutrición para modelar el IMC cual es un indicador del peso de una persona en relación con su altura, sin embargo, se consideraron factores que atañen directamente la exposición de la población a la obesidad lo cual afecta directamente el valor del IMC y por tanto la calidad de la información que éste arroja. Para el estudio se consideraron las siguientes variables:

>- IMC: El índice de masa corporal (kg/m2).
>- Sexo: 1=mujer, 0=hombre.
>- Edad: Edad en años.
>- Horas_suenio: Horas de sueño promedio.
>- Cintura: Cirfunferencia de la cintura.
>- Tam_pob: Densidad poblacional 
>- Act_fis_vig: Minutos por semana de actividad física vigorosa. 
>- Act_fis_mod: Minutos por semana de actividad física moderada
>- Act_fis_cam: Minutos por semana de caminata.
>- Act_fis: Minutos de actividad física total.
>- Tiempo_pantalla: Tiempo en pantalla en minutos.

La causa de la obesidad se remonta a que la cantidad de energía ingerida es menor que la que se consume, sin embargo, se pretende probar qué variables influyen en mayor proporción al aumento de la masa corporal.


# I: Análisis exploratorio


Comenzamos cargando los datos de la base de datos.
Incialmente se cargan los datos por separado, luego se mezclan, se cambia el nombre de columnas y utilizamos 'datos' como nuestra nueva fuente de los datos a usar.
Los datos de fumador no son suficientes y no serán considerados en el proyecto.
La variable cintura es puesta como valores numéricos para que R no lo tome como muchos niveles distintas.
Por último cambiamos la escala de los datos de actividad física, para tener en promedio cuántos minutos al día se tienen para cada actividad.
```{r datos 1, echo=FALSE, message=FALSE, warning=FALSE, paged.print=FALSE}
library(foreign)
library(nortest)
library(MPV)
library(lmtest)
library(MASS)

datos.sue <- read.spss("ensanutsue_o_16112016.sav",to.data.frame = T)
datos.act <- read.spss("ponde_actfis_20a69_2_original_trabajada_SSA.sav",to.data.frame = T)
datos.sue <- datos.sue[, c('idinsert','imc','sexofem','edad','su101','cintura','rural')]
datos.act <- datos.act[, c('idinsert','afv_s','afm_s','afw_s','afmv_sem','screen_dic')]
datos <- merge(datos.sue,datos.act,by='idinsert')
datos$idinsert <- NULL
colnames(datos) <- c('imc','sexo','edad','horas_suenio','cintura',
                     'tam_pob','act_fis_vig','act_fis_mod','act_fis_cam',
                     'act_fis','tiempo_pantalla')
rm(datos.act,datos.sue)

datos$fumador <- NULL
datos <- na.omit(datos)
datos$cintura <- as.numeric(as.character(datos$cintura))

datos$act_fis_vig <- datos$act_fis/7
datos$act_fis_cam <- datos$act_fis_cam/7
datos$act_fis_mod <- datos$act_fis_mod/7
datos$act_fis     <- datos$act_fis/7
```


Visualizamos los datos limpiados con ayuda de summary, un boxplot y un histograma.
Iniciamos con una muestra de tamaño N = 1170.
```{r datos 2, echo=FALSE, fig.width = 3.5, fig.height = 3}
summary(datos)
hist(datos$imc, xlab = "IMC", ylab = "Frecuencia", main = "Distribucion de IMC")
boxplot(datos$imc, ylab = "IMC", main = "Boxplot de IMC")
```


Al graficar los datos de hora de sueño notamos que dos valores (58 y 99) no son posibles.
```{r datos 3, echo = FALSE, fig.align = "center", fig.width = 3.5, fig.height = 3}
plot(datos$horas_suenio,datos$imc, xlab = "Horas de sueño", ylab = "IMC", main = "Datos Sueño Originales")
```

Eliminamos los dos valores que no son válidos(58 y 99)
```{r, echo = FALSE, fig.align="center", fig.width = 3.5, fig.height = 3}
index <- which(datos$horas_suenio=='58' | datos$horas_suenio=='99')
datos <- datos[-index,]

plot(datos$horas_suenio,datos$imc, xlab = "Horas de sueño", ylab = "IMC", main = "Datos Sueño Limpios")
```
Al eliminar estos dos datos tenemos una muestra de tamaño N = 1168.

Obtenemos las graficas de dispersion para otras variables continuas para evaluar si todos los datos son posibles.
```{r datos 4, echo = FALSE, fig.width = 6, fig.height = 6}
pairs(imc ~ cintura + act_fis_vig + act_fis_mod + act_fis_cam + act_fis, data = datos, main = "Variables Continuas")
```


Se tienen registrados individuos con cintura de más de 200 cm y un IMC bajo, se quitaran estos valores de la base de datos.
```{r datos 5, echo = FALSE, fig.width = 3.5, fig.height = 3, fig.align= "center"}
sort(datos$cintura, decreasing=T)[1:10]
index <- which(datos$cintura > 200)
datos <- datos[-index,]
plot(datos$cintura, datos$imc, xlab = "cintura", ylab = "IMC", main = "Datos Cintura Limpios")
```
Después de eliminar estos datos, nuestro tamaño de muestra final es N = 1160.


# II: Selección de modelo.

Ya que tenemos los datos limpios (N = 1160) buscamos las variables significativas.
```{r Selección de Modelo, include=FALSE}
# modelo mas simple (nulo)
model.base <- lm(imc~1,data = datos)
# modelo completo
model.full <- lm(imc~.,data = datos)

# seleccion forward
fwd.model  <- step(model.base,scope =list(lower=model.base,upper = model.full),
                   direction = "forward")

# seleccion backward, se comienza por el modelo completo
bwd.model  <- step(model.full,scope =list(lower=model.base,upper = model.full),
                   direction = "backward")
#seleccion en ambos sentidos, se utiliza el modelo completo
both.model  <- step(model.full,scope =list(lower=model.base,upper = model.full),
                    direction = "both")
```

Mostramos los modelos
```{r Modelos}
fwd.model
bwd.model
both.model
```

Como los métodos, backward, forward y both dan el mismo modelo, seleccionamos el modelo con variables: cintura, sexo, edad y tamaño de población.
```{r}
anova(fwd.model)
summary(fwd.model)$adj.r.squared
```


# III: Ajustes de modelos de regresión.
**Nota:** Los valores TRUE indican que para un nivel de confianza al (1 - 0.05)% el coefieciente asociado a la variable es distinto de cero. FALSE indica que el coeficiente es cero.

Ahora hacemos modelos de regresión lineal simple para todas las variables a considerar:
```{r Regresión Simple Lineal con cintura, echo=FALSE}
#Establecemos un nivel de confianza de 0.05
alpha = 0.05

modelo <- lm(imc ~ cintura, data=datos)

anova(modelo)
#Obtenemos el coeficiente de determinación:
summary(modelo)$r.squared
#Probamos el p-value de la prueba F 
anova(modelo)$Pr[1] < alpha
```
En este modelo podemos ver que se explica el **64.10938%** de la variabilidad del modelo.
Fijandonos en el p-value de la prueba F, para un nivel de significancia de 0.05, vemos que la variable cintura **sí** es significante y podemos suponer que el coefieciente asociado a la variable cintura es distinto de 0.
Entonces tenemos un modelo que explica una buena parte de la variabilidad y es significante.
Este modelo resulta ser la **mejor** regresión lineal simple, explica más la variabilidad por mucho y en la gráfica notamos que ajusta adecuadamente a los datos.


```{r Regresión Simple Lineal con horas de sueño, echo=FALSE}
#Establecemos un nivel de confianza de 0.05
alpha = 0.05

modelo1 <- lm(imc ~ horas_suenio, data=datos)

anova(modelo1)
#Obtenemos el coeficiente de determinación:
summary(modelo1)$r.squared
#Probamos el p-value de la prueba F 
anova(modelo1)$Pr[1] < alpha
```
Se observa que el 0.04% de la variabilidad del modelo está explicada.
Considerando  el p-value de la prueba F, para un nivel de significancia de 0.05, vemos que la variable horas_suenio sí resulta significante y podemos suponer que el coefieciente asociado a la variable horas de sueño es distinto de 0.
Entonces el modelo es significante. Pero no es muy bueno ya que la recta no ajusta adecuadamente y explica muy poca variabilidad.


```{r Regresión Simple Lineal con edad, echo=FALSE}
#Establecemos un nivel de confianza de 0.05
alpha = 0.05

modelo2 <- lm(imc ~ edad, data=datos)

anova(modelo2)
#Obtenemos el coeficiente de determinación:
summary(modelo2)$r.squared
#Probamos el p-value de la prueba F 
anova(modelo2)$Pr[1] < alpha
```
Se observa que el 0.03% de la variabilidad del modelo está explicada.
Considerando  el p-value de la prueba F, para un nivel de significancia de 0.05, vemos que el coefieciente asociado a la variable edad NO resulta significante, podemos suponer que el coeficiente es IGUAL a  0.
El modelo no explica la variabilidad y no es significante, por lo tanto no es un modelo a considerar.
```{r Regresión Simple Lineal con sexo, echo=FALSE}
#Establecemos un nivel de confianza de 0.05
alpha = 0.05

modelo3 <- lm(imc ~ sexo, data=datos)

anova(modelo3)
#Obtenemos el coeficiente de determinación:
summary(modelo3)$r.squared
#Probamos el p-value de la prueba F 
anova(modelo3)$Pr[1] < alpha
```
Se observa que el 3% de la variabilidad del modelo está explicada.
Considerando  el p-value de la prueba F, para un nivel de significancia de 0.05, vemos que la variable sexo  sí resulta significante y podemos suponer que el coefieciente asociado a la variable sexo es distinto de 0.
El modelo es significante, pero no es muy bueno ya que explica poca variabilidad.



```{r Regresión Simple Lineal con tamaño de población, echo=FALSE}
#Establecemos un nivel de confianza de 0.05
alpha = 0.05

modelo4 <- lm(imc ~ tam_pob, data=datos)
anova(modelo4)
#Obtenemos el coeficiente de determinación:
summary(modelo4)$r.squared
#Probamos el p-value de la prueba F 
anova(modelo4)$Pr[1] < alpha
```
Se observa que el 1.2% de la variabilidad del modelo está explicada.
Considerando  el p-value de la prueba F, para un nivel de significancia de 0.05, vemos que la variable tam_pob sí resulta significante y podemos suponer que el coefieciente asociado a la variable tam_pob es distinto de 0.
El modelo es significante, sigue sin explicar mucha variabilidad, sin embargo, explica más que las otras variables discretas.


```{r Regresión Simple Lineal con actividad física vigorosa, echo=FALSE}
#Establecemos un nivel de confianza de 0.05
alpha = 0.05

modelo5 <- lm(imc ~ act_fis_vig, data=datos)
anova(modelo5)
#Obtenemos el coeficiente de determinación:
summary(modelo5)$r.squared
#Probamos el p-value de la prueba F 
anova(modelo5)$Pr[1] < alpha
```
Se observa que el .01% de la variabilidad del modelo está explicada.
Considerando  el p-value de la prueba F, para un nivel de significancia de 0.05, vemos que la variable act_fis_vig NO resulta significante, podemos suponer que el coefieciente asociado a la variable act_fis_vig es IGUAL a 0.
Este modelo no es significante, lo que parece sorprendente, ya que se suele asociar el ejercicio vigoroso con la pérdida de peso.


```{r Regresión Simple Lineal con actividad física moderada, echo=FALSE}
#Establecemos un nivel de confianza de 0.05
alpha = 0.05

modelo6 <- lm(imc ~ act_fis_mod, data=datos)
anova(modelo6)
#Obtenemos el coeficiente de determinación:
summary(modelo6)$r.squared
#Probamos el p-value de la prueba F 
anova(modelo6)$Pr[1] < alpha
```
Se observa que el 0.04% de la variabilidad del modelo está explicada.
Considerando  el p-value de la prueba F, para un nivel de significancia de 0.05, vemos que la variable  act_fis_mod sí resulta significante y podemos suponer que el coefieciente asociado a la variable act_fis_mod es distinto de 0.
El modelo es significante, pero no muy bueno, lo que de nuevo es sorprendente, ya que sugiere que la actividad física no es la mejor manera de modelar el imc.


```{r Regresión Simple Lineal con actividad física cam, echo=FALSE}
#Establecemos un nivel de confianza de 0.05
alpha = 0.05

modelo7 <- lm(imc ~ act_fis_cam, data=datos)
anova(modelo7)
#Obtenemos el coeficiente de determinación:
summary(modelo7)$r.squared
#Probamos el p-value de la prueba F 
anova(modelo7)$Pr[1] < alpha
```
Se observa que el modelo explica casi 0% de la variabilidad.
Considerando  el p-value de la prueba F, para un nivel de significancia de 0.05, vemos que la variable Actividad Fisica Cam NO resulta significante, podemos suponer que el coefieciente asociado a la variable act_fis_cam IGUAL a  0.
El modelo es no significante, para ninguna de las variables de actividad física se tiene un buen ajuste.


```{r Regresión Simple Lineal con actividad física, echo=FALSE}
#Establecemos un nivel de confianza de 0.05
alpha = 0.05

modelo8 <- lm(imc ~ act_fis, data=datos)
anova(modelo8)
#Obtenemos el coeficiente de determinación:
summary(modelo8)$r.squared
#Probamos el p-value de la prueba F 
anova(modelo8)$Pr[1] < alpha
```
Se observa que el .01% de la variabilidad del modelo está explicada.
considerando  el p-value de la prueba F, para un nivel de significancia de 0.05, vemos que la variable Actividad Fisica NO resulta significante, podemos suponer que el coefieciente asociado a la variable act_fis IGUAL a  0.
El modelo es no significante, lo que mantiene que la actividad física no es una variable adecuada para explicar la variable respuesta de imc.


```{r Regresión Simple Lineal con tiempo en pantalla, echo=FALSE}
#Establecemos un nivel de confianza de 0.05
alpha = 0.05

modelo9 <- lm(imc ~ tiempo_pantalla, data=datos)
anova(modelo9)
#Obtenemos el coeficiente de determinación:
summary(modelo9)$r.squared
#Probamos el p-value de la prueba F 
anova(modelo9)$Pr[1] < alpha
```
Se observa que el .001% de la variabilidad del modelo está explicada.
Considerando  el p-value de la prueba F, para un nivel de significancia de 0.05, vemos que la variable tiempo_pantalla NO resulta significante, podemos suponer que el coefieciente asociado a la variable tiempo_pantalla IGUAL a  0
El modelo es no significante, siguiendo de que es una variable discreta y no han demostrado mucha efectividad en explicar el imc.

Por lo tanto concluimos que el mejor modelo de regresión lineal simple para nuestros datos de tamaño N = 1160 es el que utiliza a la variable cintura.


## Regresión Lineal Múltiple

Ahora creamos los modelos de regresión lineal múltiple, enfocandonos en las variables que ya vimos tienen significancia en el modelo, como lo son: cintura, sexo, tam_pob y edad

Dado que cintura es la variable que mejor ajusta el modelo, empezamos buscando términos de interacción entre esta y las otras 3 variables elegidas.

Primero vemos la interacción cintura ~ edad
```{r Cintura:edad}
#Establecemos un nivel de confianza de 0.05
alpha = 0.05

modelo1 <- lm(imc ~ cintura + edad + cintura:edad, data=datos)  
anova(modelo1)

# Obtenemos el coeficiente de determinación:
summary(modelo1)$r.squared
summary(modelo1)$adj.r.squared   #R^2 adj
```


Después la interacción cintura ~ sexo
```{r cintura:sexo}
#Establecemos un nivel de confianza de 0.05
alpha = 0.05

modelo2 <- lm(imc ~ cintura + sexo + cintura:sexo, data=datos)  
anova(modelo2)

# Obtenemos el coeficiente de determinación
summary(modelo2)$r.squared
summary(modelo2)$adj.r.squared   #R^2 adj
```


La interacción tam_pob ~ cintura
```{r cintura: tam_pob}
modelo3 <- lm(imc ~ tam_pob + cintura + cintura:tam_pob, data=datos)  
anova(modelo3)

# Obtenemos el coeficiente de determinación
summary(modelo3)$r.squared
summary(modelo3)$adj.r.squared   #R^2 adj
```


Ahora probamos el modelo con las 4 variables más significativas y lo usaremos para contrastar con los términos de interacción
```{r variables significativas}
modelo4 <- lm(imc ~ cintura + sexo + edad + tam_pob, data=datos)  
anova(modelo4)
summary(modelo4)
```
Nuestro coeficiente de correlación ajustado es de 67.59973%, en los modelos siguientes veremos si podemos explicar más variabilidad.


Eliminamos el intercepto del modelo.
```{r variables significativas sin intercepto}
modelo4 <- lm(imc ~ cintura + sexo + edad + tam_pob - 1, data=datos)  
anova(modelo5)
summary(modelo5)
```
Al eliminar el intercepto se explica mucha más variabilidad, ahora explica el 98.79% de esta, sin embargo la variable tam_pobRural no es significante en este modelo.


Agregamos términos de interacción a este nuevo modelo sin intercepto
```{r cintura, sexo, edad , tam_pob , cintura:tam_pob }
modelo6 <- lm(imc ~ cintura + sexo + edad + tam_pob + cintura:tam_pob - 1, data=datos)  
anova(modelo6)
summary(modelo6)
```
Al agregar el término de interacción entre cintura y tam_pob, se obtiene una pequeña mejora en el coefieciente de correlación, sin embargo tam_pobRural no es significativo.


Cambiamos el término de interacción por cintura:sexo.
```{r sexo, edad, tam_pob, cintura:tam_pob}
modelo7 <- lm(imc ~ cintura + sexo + edad + tam_pob + cintura:sexo - 1, data=datos)  
anova(modelo7)
summary(modelo7)
```
Notamos que al cambiar el termino de interacción cintura:tam_pob por cintura:sexo, el coeficiente de correlación (ajustado y normal) disminuye y las variables cintura:sexo, tam_pobUrbano y sexo no son significativas.


```{r cintura, sexo, edad, tam_pob, cintura:edad}
modelo8 <- lm(imc ~ cintura + sexo + edad + tam_pob + cintura:edad - 1, data=datos)  
anova(modelo8)
summary(modelo8)
```
En este modelo todas nuestras variables son significativas, además al tener explicar 98.79% de la variabilidad, explica la misma cantidad que el modelo6 con término de interacción cintura:tam_pob, que tambien explica el 98.79% de la variabilidad, pero el modelo6 tiene variables no significativas.


Utilizamos el criterio de la información de Akaike para encontrar el modelo más parsimonioso.
```{r Criterio de la Información de Akaike}
AIC(modelo1, modelo2, modelo3, modelo4, modelo5, modelo6, modelo7, modelo8)
```
Vemos que el modelo6(cintura + sexo + edad + tam_pob + cintura:tam_pob - 1) tiene la menor Información de Akaike, sin embargo el modelo8(imc ~ cintura + sexo + edad + tam_pob + cintura:edad - 1) tiene casi la misma información de Akaike y todas sus variables son significativas.

Como el modelo8(cintura + sexo + edad + tam_pob + cintura:edad - 1) tiene una información de Akaike(6014.666) muy pequeña, explica la mayor cantidad de la variabilidad (Coeficiente de Determinación ajustado = 0.9879) y todas sus variables son significativas se elige a esto como el mejor modelo para estos datos con tamaño de muestra N = 1160.


##Prueba de significancia del modelo completo con términos de interacción


Probamos la significancia de la regresión del mejor modelo de regresión lineal múltiple(modelo6):
$$H_0: \beta_1 = \beta_2 = \beta_3 = \beta_4 = \beta_5 = \beta_6 = 0\quad vs\quad H_1:\beta_j \neq 0$$

> - $\beta_1$ es el coeficiente asociado a la variable cintura,
> - $\beta_2$ es el coeficiente asociado a la variable sexo.
> - $\beta_3$ es el coeficiente asociado a la variable edad.
> - $\beta_4$ es el coeficiente asociado a la variable tam_pobRural.
> - $\beta_5$ es el coeficiente asociado a la variable tam_pobUrbana.
> - $\beta_6$ es el coeficiente asociado a la variable cintura:edad.

```{r Prueba de Significancia del modelo}
#Establecemos un nivel de confianza de 0.05
alpha = 0.05

SCE <- sum((datos$imc - fitted(modelo8))^2)
SCR <- sum((fitted(modelo8) - mean(datos$imc))^2)

F <- (SCR/(6))/(SCE/1154)
F > qf(1 - alpha, 1154, 6)
```
Como el estadístico F es mayor al percentil 0.95 de una distribución F, para un nivel de confianza de 0.05 decimos que la regresión es significante.


##Detalles del modelo seleccionado


```{r Detalles}
anova(modelo8)
summary(modelo8)
```


#Diagnóstico del modelo seleccionado.


Graficamos los residuales.
```{r, echo=FALSE, fig.width = 3.5, fig.height = 2.5, fig.align="center"}
residuos<-resid(modelo8)
ygorro<-fitted(modelo8)

qqnorm(residuos,xlab="Residuos",ylab="Valores Esperados",main="Probabilidad Normal",pch=19)
qqline(residuos,col=3)
```

```{r, echo=FALSE, fig.width = 3.5, fig.height = 2.5, fig.align="center"}
hist(residuos,col=rainbow(5),border="white",main="Histograma de los residuos",xlab="Residuos",ylab="Frecuencia Relativa",col.main="darkblue",col.axis="black",col.lab="darkgreen",prob=TRUE,ylim=c(0,.5))
curve(dnorm(x,mean(residuos),sd(residuos)),add=T,col="1")
```

Realizamos la prueba de Anderson-Darling para cerificar la bondad de ajuste
```{r, echo=FALSE, fig.width = 3.5, fig.height = 3, fig.align="center"}
boxplot(residuos,xlab="Residuos",main="Box-Plot de los residuos",col.main="darkblue",col.lab="darkgreen")
ad.test(rnorm(rstandard(modelo),mean=0,sd=1)) 
```
Como el p-value de la prueba de Anderson-Darling es mayor que un nivel de confianza 0.05, decimos que nuestros errores se distribuyen normal(0, 1).


Buscamos datos influyentes mediante la distancia de Cook
```{r Distancia de Cook, fig.width = 4, fig.height = 3, fig.align="center"}
modelo8 <- lm(imc ~ sexo + edad + tam_pob + cintura:tam_pob, data=datos)
n <- dim(datos)[1]
cooksd <- cooks.distance(modelo6)
as.numeric(names(cooksd)[(cooksd > (4/n))])
```
Resulta haber varios datos influyentes, sin embargo todos parecen ser posibles.


```{r}
n <- dim(datos)[1]
influential <- which(cooksd > (4/n))
datos[influential,]$edad
datos[influential,]$act_fis
```
Son datos de personas que son de edades mayores o personas que hacen muy poco o mucho ejercicio.


Revisamos la multicolinealidad usando el Vif
```{r Vif}
Vif <- 1/(1 - summary(modelo8)$r.squared)
Vif
Vif < 10
```
Como el Vif < 10 no existe ninguna correlación significante.


Probamos la Homocedasticidad.
```{r, echo=FALSE, fig.width = 3.5, fig.height = 3, fig.align= "center"}
plot(ygorro,rstandard(modelo),xlab="Valores Ajustados",ylab="Residuos Estandarizados",main="Estandarizados vs Ajustados",pch=19,ylim=c(-4,4))
abline(h=0,col="blue",lty=3)
abline(h=c(-2,2),col="darkred",lty=2)
abline(h=c(-3,3),col="green",lty=3)
```
```{r, echo=FALSE, fig.width = 3.5, fig.height = 3, fig.align= "center"}
estudentizados <- stdres(modelo8)

plot(ygorro,estudentizados ,xlab="Valores Ajustados",ylab="Residuos Estudentizados",main="Estudentizados vs Ajustados",pch=19,ylim=c(-4,4))
abline(h=0,col="blue",lty=3)
abline(h=c(-2,2),col="darkred",lty=2)
abline(h=c(-3,3),col="green",lty=3)
```


# Conclusión
El mejor modelo, es el modelo8 de la sección de regresión lineal múltiple de este proyecto, este modelo, que lleva las variables, cintura, sexo, edad, tam_pob y la variable de interacción tam_pob:edad resulto ser la mejor forma de explicar la variable respuesta de imc.
Nos pareció importante notar que con estos datos la actividad física no parece tener un rol importante en la predicción del imc.
En nuestra experiencia tambien nos parecio sorprendente que al cambiar la variable cintura por un término de interacción con edad obtuviesemos un mejor modelo, esperabamos que la interacción con sexo tuviera un peso más importante, pero no fue así.


# Nota Técnica
A lo largo del código hay varios TRUE que aparecen, estos indican si las regresiones son significantes o no.
Usamos las librerías: foreign, MASS, MPV y lmtest.
Decidimos entregar un trabajo elaborado en Markdown para poder mostrar con agilidad los resultados junto con nuestras interpretaciones y comentarios.

## Bibliografía
University of Sheffield,Ellen Marshall and Sofia Maria Karadimitriou, https://www.sheffield.ac.uk/polopoly_fs/1.536483!/file/MASH_multiple_regression_R.pdf, 16/05/2018

