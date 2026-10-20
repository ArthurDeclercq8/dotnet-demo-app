properties([
    pipelineTriggers([pollSCM('* * * * *')])
])

node {
    stage('Checkout') {
        checkout scm
    }
    stage('Preparation') {
        sh '''
            docker rm -f todoapp || true
            docker network create todonet || true
            docker volume create mariadb-data
        '''
    }
    stage('Database') {
        sh '''
            docker inspect todoappdb >/dev/null 2>&1 || docker run -d --name todoappdb \
              --network todonet \
              -e MARIADB_ROOT_PASSWORD=sekrit \
              -e MARIADB_DATABASE=todo_db \
              -e MARIADB_USER=todo_usr \
              -e MARIADB_PASSWORD=letmeinplz \
              -v mariadb-data:/var/lib/mysql \
              mariadb:11
            for i in $(seq 1 30); do
              docker exec todoappdb healthcheck.sh --connect --innodb_initialized && break
              sleep 2
            done
            docker exec -i todoappdb mariadb -utodo_usr -pletmeinplz todo_db < TodoApp/schema.sql
        '''
    }
    stage('Build') {
        sh 'docker build -t todoapp TodoApp'
    }
    stage('Deploy') {
        sh '''
            docker run -d --name todoapp --network todonet -p 8081:8080 \
              -e "ConnectionStrings__TodoDb=Server=todoappdb;Port=3306;Database=todo_db;User=todo_usr;Password=letmeinplz;" \
              -e ASPNETCORE_ENVIRONMENT=Development \
              todoapp
        '''
    }
    stage('Results') {
        sh '''
            for i in $(seq 1 15); do
              curl -sf http://172.16.0.10:8081/ > /dev/null && exit 0
              sleep 2
            done
            echo "App antwoordt niet op poort 8081"
            exit 1
        '''
    }
}