

docker-compose up -d php

docker exec -it source-analysis-laravel /bin/bash

// composer create-project --prefer-dist laravel/laravel laravel "5.8.*"

cd laravel/

php artisan serve --host 0.0.0.0



