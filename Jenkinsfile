def server_tag = "caicloud/cyclone-server:${env.BUILD_NUMBER}"
def worker_tag = "caicloud/cyclone-worker:${env.BUILD_NUMBER}"
def registry = "cargo.caicloudprivatetest.com"

podTemplate(
    cloud: 'dev-cluster',
    namespace: 'kube-system',
    name: 'cyclone',
    label: 'cyclone',
    idleMinutes: 1440,
    containers: [
        // jnlp with kubectl
        containerTemplate(
            name: 'jnlp',
            alwaysPullImage: true,
            image: 'cargo.caicloud.io/circle/jnlp:2.62',
            command: '',
            args: '${computer.jnlpmac} ${computer.name}',
        ),
        // docker in docker
        containerTemplate(
            name: 'dind', 
            image: 'cargo.caicloud.io/caicloud/docker:17.03-dind', 
            ttyEnabled: true, 
            command: '', 
            args: '--host=unix:///home/jenkins/docker.sock',
            privileged: true,
        ),
        // golang with docker client
        containerTemplate(
            name: 'golang',
            image: 'cargo.caicloud.io/caicloud/golang-docker:1.8-17.03',
            ttyEnabled: true,
            command: '',
            args: '',
            envVars: [
                containerEnvVar(key: 'DEBUG', value: 'true'),
                containerEnvVar(key: 'CLAIR_DISABLE', value: 'true'),
                containerEnvVar(key: 'MONGODB_HOST', value: '127.0.0.1:27017'),
                containerEnvVar(key: 'KAFKA_HOST', value: '127.0.0.1:9092'),
                containerEnvVar(key: 'ETCD_HOST', value: 'http://127.0.0.1:2379'),
                containerEnvVar(key: 'REGISTRY_LOCATION', value: 'cargo.caicloud.io'),
                containerEnvVar(key: 'REGISTRY_USERNAME', value: 'caicloudadmin'),
                containerEnvVar(key: 'REGISTRY_PASSWORD', value: 'caicloudadmin'),
                containerEnvVar(key: 'WORKER_IMAGE', value: "${registry}/${worker_tag}"),
                containerEnvVar(key: 'DOCKER_HOST', value: 'unix:///home/jenkins/docker.sock'),
                containerEnvVar(key: 'DOCKER_API_VERSION', value: '1.26'),
                containerEnvVar(key: 'WORKDIR', value: '/go/src/github.com/caicloud/cyclone')
            ],
        ),
         containerTemplate(
            name: 'zk',
            image: 'cargo.caicloud.io/caicloud/zookeeper:3.4.6',
            ttyEnabled: true,
            command: "",
            args: "",
        ),
         containerTemplate(
            name: 'kafka',
            image: 'cargo.caicloud.io/caicloud/kafka:0.10.1.0',
            ttyEnabled: true,
            command: '',
            args: '',
            envVars: [
                containerEnvVar(key: 'KAFKA_ADVERTISED_HOST_NAME', value: '0.0.0.0'),
                containerEnvVar(key: 'KAFKA_ADVERTISED_PORT', value: '9092'),
                containerEnvVar(key: 'KAFKA_ZOOKEEPER_CONNECT', value: 'localhost:2181'),
                containerEnvVar(key: 'KAFKA_ZOOKEEPER_CONNECTION_TIMEOUT_MS', value: '60000'),
            ],
        ),
        containerTemplate(
            name: 'mongo',
            image: 'cargo.caicloud.io/caicloud/mongo:3.0.5',
            ttyEnabled: true,
            command: 'mongod',
            args: '--smallfiles',
        ),
        containerTemplate(
            name: 'etcd',
            image: 'cargo.caicloud.io/caicloud/etcd:v3.1.3',
            ttyEnabled: true,
            command: 'etcd',
            args: '-name=etcd0 -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379 -initial-advertise-peer-urls http://127.0.0.1:2380 -listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster-1 -initial-cluster etcd0=http://127.0.0.1:2380 -initial-cluster-state new',
        ),
    ]
) {
    node('cyclone') {
        stage('Checkout') {
            checkout scm
        }
        container('golang') {
            ansiColor('xterm') {

                stage("Complie") {
                    sh('''
                        set -e 
                        mkdir -p $(dirname ${WORKDIR}) && ln -sf $(pwd) ${WORKDIR}
                        cd ${WORKDIR}

                        echo "buiding server"
                        go build -i -v -o cyclone-server github.com/caicloud/cyclone/cmd/server

                        echo "buiding worker"
                        go build -i -v -o cyclone-worker github.com/caicloud/cyclone/cmd/worker 
                        docker build -t ${WORKER_IMAGE} -f Dockerfile.worker .
                    ''')
                }

                stage('Run e2e test') {
                    sh('''
                        set -e
                        cd ${WORKDIR}
                        # get host ip
                        HOST_IP=$(ifconfig eth0 | grep 'inet addr:'| grep -v '127.0.0.1' | cut -d: -f2 | awk '{ print $1}')
                        export CYCLONE_SERVER=http://${HOST_IP}:7099
                        export LOG_SERVER=ws://${HOST_IP}:8000/ws
                        
                        # kill cyclone server process left by last build
                        cyclone_pid=$(ps -ef | grep cyclone-server | grep -v "grep" | awk '{print $1}')
                        if [[ -n "${cyclone_pid}" ]]; then
                            kill -9 ${cyclone_pid}
                        fi

                        echo "start server"
                        ./cyclone-server --cloud-auto-discovery=false --log-force-color=true &

                        echo "testing ..."
                        # go test compile
                        go test -i ./tests/...
                        
                        # go test
                        go test -v ./tests/service 
                        go test -v ./tests/version 
                        go test -v ./tests/yaml
                    ''')
                }
            }

            stage("Build image and publish") {
                sh "docker build -t ${server_tag} -f Dockerfile.server ."
                sh "docker build -t ${worker_tag} -f Dockerfile.worker ."

                docker.withRegistry("https://${registry}", "cargo-private-admin") {
                    docker.image(server_tag).push()
                    docker.image(worker_tag).push()
                }
            }
        }

        stage("deploy") {
            sh("""
                kubectl --namespace cyclone get deploy circle-server-v0.0.1 -o yaml | sed 's/cyclone-server:.*\$/cyclone-server:${env.BUILD_NUMBER}/; s/cyclone-worker:.*\$/cyclone-worker:${env.BUILD_NUMBER}/' | kubectl --namespace cyclone replace -f -
            """)
        }
    }
}
