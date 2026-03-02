# WIN 24 🏀⚽️

![Logo WIN24](./project-docs/WIN24%20Logo.png)

## Grupo Bolirana 🐸

## Contacto e integrantes
- Samuel Ricardo Pardo Trujillo - spardotr@unal.edu.co
- Daniel Santiago Rincón Santofimio - danriconsa@unal.edu.co
- Camilo Andrés Salinas Cuervo - csalinasc@unal.edu.co
- Juan Pablo Zuluaga Galindo - jzuluagaga@unal.edu.co

## De qué trata
La idea nació de algo sencillo: ¿cómo funcionan realmente las casas de apuestas? Queríamos entenderle tanto al negocio como a la parte técnica, y de ahí salió WIN 24.

Básicamente, los usuarios pueden apostar en eventos deportivos (fútbol). El admin crea los eventos con ayuda de unas APIs que traen info real de equipos y cuotas de referencia, pero él define las cuotas finales.

Lo diferente es que tenemos un motor de riesgo. Mientras la gente apuesta, el sistema va calculando cuánto tendría que pagar la casa si gana cada resultado. Si el riesgo pasa cierto límite, le avisa al admin y le sugiere cambiar las cuotas para equilibrar la cosa. El admin decide si las cambia o no.

Cuando termina el evento, el admin mete el resultado y el sistema liquida todo automáticamente: las apuestas ganadoras se marcan como "GANADA" y se les acredita la plata a los usuarios.

Estamos armándolo con Spring Boot, React y MySQL. Las APIs que usamos son TheSportsDB para datos de equipos y eventos, y The Odds API para las cuotas de referencia.
