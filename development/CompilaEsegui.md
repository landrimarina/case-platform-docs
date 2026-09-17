nuova app FE:
    cd /home/marinalandri/workspace/case-platform-anc/frontend
    nvm use 24
    npm install     
Sviluppo: npm run dev
Compilazione di produzione: npm run build (genera dist/)


Nuova app BE:
    cd /home/marinalandri/workspace/case-platform-anc/backend
    export JAVA_HOME=/opt/java/jdk-21.0.11+10
    export PATH="$JAVA_HOME/bin:$PATH"
    ./mvnw clean compile          # solo compilazione
    ./mvnw quarkus:dev e testare /api/anc/home?