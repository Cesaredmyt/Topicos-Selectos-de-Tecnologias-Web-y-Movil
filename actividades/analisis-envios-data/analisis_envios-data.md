# Análisis de envios-data

# Análisis por archivo

| Archivo | Moto  | Camioneta | Bicicleta | Dron  | Eskuter |
| ------- | ----- | --------- | --------- | ----- | ------- |
| envios1 | 25.0% | 25.0%     | 25.0%     | 25.0% | —       |
| envios2 | 34.7% | 14.4%     | 26.2%     | 8.6%  | 16.2%   |
| envios3 | 73.7% | 15.8%     | 7.7%      | 2.8%  | —       |
| envios4 | 57.2% | 16.9%     | 18.8%     | 7.1%  | —       |
| envios5 | 45.8% | 16.8%     | 28.0%     | 9.5%  | —       |
| envios6 | 63.1% | 31.9%     | 4.2%      | 0.7%  | —       |

- **envios1:** perfectamente balanceado, 2,500 envíos por medio.
- **envios2:** es el único con `eskuter`, que representa el 16.2%. El modelo no lo conoce.
- **envios3:** domina la moto (73.7%) y el dron es muy escaso (2.8%).
- **envios4 y envios5:** la moto domina de forma moderada. En envios5 la bicicleta tiene una presencia mayor (28.0%).
- **envios6:** moto y camioneta suman cerca del 95%. Solo hay 73 drones (0.7%) y 424 bicicletas (4.2%).

# Predicción:

1. `envios1` debería tener buena exactitud por estar balanceado, aunque el balance por sí solo no garantiza que sea la mejor.
2. El `eskuter` no se podrá predecir, porque el modelo nunca lo vio en el entrenamiento.
3. `envios2` debería ser el archivo con peor exactitud, ya que el 16% de sus envíos (más de 1,600) pertenece a un medio desconocido.
4. Los envíos de `eskuter` deberían confundirse con los medios más parecidos: moto o bicicleta.
5. El dron debería ser el medio más difícil de acertar, porque tiene pocos ejemplos en la mayoría de los archivos.
6. En `envios6` el dron tiene muy pocos casos (73), por lo que su resultado será poco confiable.
7. En los archivos dominados por moto (`envios3` y `envios6`) la exactitud puede verse alta aunque el modelo falle en los medios minoritarios.

# Resultados de la prueba con el árbol de decisión

## Resumen

La columna "Siempre la mayoritaria" indica la exactitud que se lograría respondiendo siempre con el medio más frecuente del archivo.

| Archivo | Exactitud | Siempre la mayoritaria | Error más notable                                        |
| ------- | --------- | ---------------------- | -------------------------------------------------------- |
| envios1 | 95.4%     | 25.0%                  | 453 drones predichos como bicicleta                      |
| envios2 | 77.7%     | 34.7%                  | 1,616 eskuter fallados (1,405 a moto, 177 a bicicleta)   |
| envios3 | 94.5%     | 73.7%                  | 204 camionetas predichas como moto                       |
| envios4 | 92.7%     | 57.2%                  | 188 camionetas y 134 bicicletas predichas como moto      |
| envios5 | 92.5%     | 45.8%                  | 171 motos predichas como bicicleta                       |
| envios6 | 96.2%     | 63.1%                  | dron con precisión de 58% (43 motos predichas como dron) |

## ¿Se cumplieron las expectativas?

1. **`envios1` con buena exactitud: se cumplió, pero no fue la mejor.** Obtuvo 95.4%. La mejor fue `envios6` con 96.2%, a pesar de ser de los más desbalanceados.
2. **`eskuter` no se predice: se cumplió.** Precisión y recall de 0.
3. **`envios2` con la peor exactitud: se cumplió.** Obtuvo 77.7%, muy por debajo de los demás (92.5% a 96.2%).
4. **`eskuter` confundido con moto o bicicleta: se cumplió.** De 1,616 envíos, 1,405 se predijeron como moto (87%) y 177 como bicicleta (11%). Los otros 34 se predijeron como dron.
5. **El dron es el medio más difícil: se cumplió.** Tiene el menor f1 de los cuatro medios conocidos en cinco de los seis archivos. La excepción es `envios2`, donde el menor es el `eskuter`.
6. **Dron poco confiable en `envios6`: se cumplió.** Se acertaron 61 de 73 (recall de 84%), pero la precisión es de 58% porque 43 motos se predijeron como dron. Con tan pocos casos, la cifra cambia mucho con pocos errores.
7. **Exactitud engañosa con moto dominante: se cumplió en parte.** En `envios3` la exactitud es 94.5% y la moto es 73.7% del archivo, pero el dron solo logra 62.8% de precisión. La exactitud oculta ese problema. Aun así, el modelo supera siempre a la regla de responder "moto", así que no se limita a repetir el medio más común.

## Patrones que se repiten

- **La moto recibe la mayoría de los errores.** La mayoría de las camionetas y de los eskuters mal clasificados terminan como moto. Es el medio más común y el más parecido a otros.
- **El dron se confunde con la bicicleta.** Ocurre en todos los archivos y es más fuerte en `envios1` (453 casos). Ambos se usan para paquetes ligeros y distancias cortas.
- **La moto y la bicicleta se confunden entre sí.** En `envios2` a `envios6` se cuentan entre 42 y 171 errores en cada dirección.
- **Moto predicha como dron.** Aunque son pocos casos por archivo, son suficientes para bajar la precisión del dron cuando este es escaso: 62.8% en `envios3` y 58.1% en `envios6`.
- **La camioneta casi nunca se predice por error.** Su precisión es de 99% o más en todos los archivos. Su recall es menor (85% a 93%) porque algunas camionetas reales se predicen como moto.

## Conclusiones

- En los cinco archivos que contienen solo los cuatro medios conocidos, la exactitud va de 92.5% a 96.2%. En todos supera ampliamente la regla de responder siempre el medio más común.
- Un medio que el modelo no vio en el entrenamiento no se puede predecir. En `envios2` el `eskuter` falló al 100% y bajó la exactitud a 77.7%. Si un medio nuevo se incorpora, el modelo debe volver a entrenarse.
- La exactitud sola no basta. Hay que revisar el resultado por medio, sobre todo cuando un archivo está dominado por uno solo.
- Los medios menos frecuentes (dron y bicicleta) son los menos estables. Cuando hay pocos casos, unos cuantos errores cambian mucho el resultado.
- El modelo reproduce los patrones de las etiquetas de los CSV. Que acierte no garantiza que la recomendación sea la mejor en la práctica, solo que coincide con lo que dicen los datos.
