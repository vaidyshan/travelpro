# Travelpro - Travel the world

## Run with Docker

Build the image:

```bash
docker build -t travelpro:local .
```

Run the application on port 8080:

```bash
docker run --name travelpro -p 8080:80 travelpro:local
```

Open <http://localhost:8080>.
