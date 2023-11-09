# SUBMISSION : Who Rule The World app? CATs or DOGs ?

## Tasks
1. Fork the repository
2. Create Dockerfiles for each containers, worker, vote, seed-data and result.
3. At the root create the docker-compose.build.yml file: 
```
version: '3' 
services:
	worker:
		build:
			context: ./worker 
		networks: 
		 - registry-network 
	vote:
		build:
			context: ./vote 
		networks: 
		 - registry-network 
	seed-data: 
		build:
			context: ./seed-data 
		networks: 
		 - registry-network 
	result:
		build:
			context: ./result 
		networks:
		 - registry-network 

networks:
	registry-network: 
		external: true
```
![Docker compose build file](docker-compose-build.png)
4. Tag and publish on my public registry:

```
sudo docker tag who-rule-the-world-worker fairouzbf/who-rule-the-world-worker
sudo docker tag who-rule-the-world-result fairouzbf/who-rule-the-world-result
sudo docker tag who-rule-the-world-vote fairouzbf/who-rule-the-world-vote
sudo docker tag who-rule-the-world-seed-data fairouzbf/who-rule-the-world-seed-data
```
![Tag](tag.png)
![Public](public.png)
![Publish](publish.png)

5. Write the compose.yml file:
```
services:
	worker:
		image: localhost:5000/worker
		depends_on:
			redis:
				condition: service_healthy
			db:
				condition: service_healthy
		networks:
			- back-tier

	vote:
		image: localhost:5000/vote
		ports:
			- "5002:80"
		networks:
			- front-tier
			- back-tier
		healthcheck:
			test: ["CMD", "curl", "-f", "http://localhost"]
			interval: 15s
			timeout: 5s
			retries: 3
			start_period: 10s
		volumes:
      - ./vote:/usr/local/app
	
	seed-data:
    image: localhost:5000/seed-data
    profiles: ["seed"]
    depends_on:
      vote:
        condition: service_healthy
    restart: "no"
    networks: 
      - front-tier

	result:
		image: localhost:5000/result
		volumes:
			- ./result:/usr/local/app
		ports:
			- "5001:80"
			- "127.0.0.1:9229:9229"
		networks:
			- back-tier
		entrypoint: nodemon --inspect=0.0.0.0 server.js
		depends_on:
			db:
        condition: service_healthy
	
	  db:
    image: postgres:15-alpine
    networks: 
      - back-tier
    volumes:
      - "db-data:/var/lib/postgresql/data"
      - "./healthchecks:/healthchecks"
    healthcheck:
      test: /healthchecks/postgres.sh
      interval: "5s"
    environment:
      POSTGRES_PASSWORD: "postgres"
      POSTGRES_USER: "postgres"
      POSTGRES_DB: "postgres"
    ports:
      - 5432:5432
    
  redis:
    image: redis
    volumes:
      - "./healthchecks:/healthchecks"
    networks: 
      - back-tier
    healthcheck:
      test: /healthchecks/redis.sh
      interval: "5s"

networks:
	front-tier:
	back-tier:

volumes:
  db-data:
```
