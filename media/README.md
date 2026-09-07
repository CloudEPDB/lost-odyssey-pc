# Capturas

Las capturas son lo que vende un repositorio como este. La gente mira las
imágenes, decide en cinco segundos si el proyecto va en serio, y solo después
lee. Merece la pena tomarlas con cuidado.

## Las que hacen falta

Por orden de importancia:

1. **Comparativa 720p vs 1080p nativo** — el mismo plano exacto, mismo sitio,
   mismo momento. Es *la* imagen del proyecto: se ve que el HUD y los menús
   crecen con la resolución, no solo el 3D. Es lo que nadie más tiene.
2. **El menú de opciones (F2)** sobre el juego corriendo. Demuestra que esto es
   un port con interfaz propia, no un parche suelto.
3. **Un plano bonito del juego a 4K** con anisotrópico y supersampling. Vale una
   cinemática o un exterior amplio.
4. **Antes / después de un parche visible** — el parpadeo de personajes o el
   motion blur.
5. **El pack de texturas**, si hay alguna textura sustituida que se note.

## Cómo tomarlas

Con el juego abierto y colocado donde quieras, hay un script que captura la
ventana una sola vez (`capture.ps1`, en el directorio de trabajo). Hace falta
traer la ventana al frente: leer directamente del buffer devuelve negro con
superficies D3D12 y Vulkan.

Para la comparativa 720p/1080p, lo importante es no mover la cámara entre las
dos: guarda la partida en el punto elegido, captura, cambia el preset en el F2
(relanza solo), carga y captura otra vez desde el mismo sitio.

## Formato

- PNG, sin recortar ni reescalar.
- Nombres descriptivos: `1080p-nativo-menu.png`, `720p-comparativa.png`.
- Si alguna pesa demasiado para el repositorio, mejor reducirla de tamaño que
  pasarla a JPEG: los artefactos se notan justo en lo que se quiere enseñar.
