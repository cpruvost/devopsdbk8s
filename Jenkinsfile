pipeline {
 
    agent {
		docker { 
			image 'cpruvost/infraascode:latest'
			args '-u root:root'
		}
	}
	
	//Parameters of the pipeline. You can define more parameters in this pipeline in order to have less hard code variables.
	parameters {
        password(defaultValue: "xxxxxx", description: 'What is the vault token ?', name: 'VAULT_TOKEN')
		string(defaultValue: "130.61.125.123", description: 'What is the vault server IP Address ?', name: 'VAULT_SERVER_IP')
		string(defaultValue: "demoatp", description: 'What is the vault secret name ?', name: 'VAULT_SECRET_NAME')  
		string(defaultValue: "atpdb2", description: 'What is the database name ?', name: 'DATABASE_NAME') 
		password(defaultValue: "AlphA_2014_!", description: 'What is the database password ?', name: 'DATABASE_PASSWORD')  		
		choice(name: 'CHOICE', choices: ['Create', 'Remove'], description: 'Choose between Create or Remove Infrastructure')
    }
	
	//Load the parameters as environment variables
	environment {
		//Vault Env mandatory variables
		VAULT_TOKEN = "${params.VAULT_TOKEN}"
		VAULT_SERVER_IP = "${params.VAULT_SERVER_IP}"
		VAULT_ADDR = "http://${params.VAULT_SERVER_IP}:8200"
		VAULT_SECRET_NAME = "${params.VAULT_SECRET_NAME}"
		CHOICE = "${params.CHOICE}"
		
		//Database variables
		TF_VAR_autonomous_database_db_name = "${params.DATABASE_NAME}"
		TF_VAR_autonomous_database_db_password = "${params.DATABASE_PASSWORD}"
		
		//Sqlcl env variables for sqlcl oci option.
		TNS_ADMIN = "./"

	}
    
    stages {
		stage('Display User Name') {
            steps {
			    wrap([$class:'BuildUser']) {
				    echo "${BUILD_USER}"
				}
            }
        }
	
        stage('Check Infra As Code Tools') {
            steps {
				sh 'whoami'
				sh 'pwd'
				sh 'ls'
			    sh 'chmod +x ./showtoolsversion.sh'
                sh './showtoolsversion.sh'
            }
        }
		
		stage('Init Cloud Env Variables') {
            steps {
				script {
					//Get all cloud information.
					env.TF_VAR_tenancy_ocid = sh returnStdout: true, script: 'vault kv get -field=tenancy_ocid secret/demoatp'
					env.TF_VAR_user_ocid = sh returnStdout: true, script: 'vault kv get -field=user_ocid secret/demoatp'
					env.TF_VAR_fingerprint = sh returnStdout: true, script: 'vault kv get -field=fingerprint secret/demoatp'
					env.TF_VAR_compartment_ocid = sh returnStdout: true, script: 'vault kv get -field=compartment_ocid secret/demoatp'
					env.TF_VAR_region = sh returnStdout: true, script: 'vault kv get -field=region secret/demoatp'
					env.DOCKERHUB_USERNAME = sh returnStdout: true, script: 'vault kv get -field=dockerhub_username secret/demoatp'
					env.DOCKERHUB_PASSWORD = sh returnStdout: true, script: 'vault kv get -field=dockerhub_password secret/demoatp'
					env.KUBECONFIG = './kubeconfig'
					
					//Terraform debugg option if problem
					//env.TF_LOG="DEBUG"
					//env.OCI_GO_SDK_DEBUG="v"
				}
				
				//Check all cloud information.
				echo "TF_VAR_tenancy_ocid=${TF_VAR_tenancy_ocid}"
				echo "TF_VAR_user_ocid=${TF_VAR_user_ocid}"
				echo "TF_VAR_fingerprint=${TF_VAR_fingerprint}"
				echo "TF_VAR_compartment_ocid=${TF_VAR_compartment_ocid}"
				echo "TF_VAR_region=${TF_VAR_region}"
				echo "DOCKERHUB_USERNAME=${DOCKERHUB_USERNAME}"
				echo "DOCKERHUB_PASSWORD=${DOCKERHUB_PASSWORD}"
				echo "KUBECONFIG=${KUBECONFIG}"
				
				
				script {
					//Get the API and SSH encoded key Files with vault client because curl breaks the end line of the key file
					sh 'vault kv get -field=api_private_key secret/demoatp | tr -d "\n" | base64 --decode > bmcs_api_key.pem'
					sh 'vault kv get -field=ssh_private_key secret/demoatp | tr -d "\n" | base64 --decode > id_rsa'
					sh 'vault kv get -field=ssh_public_key secret/demoatp | tr -d "\n" | base64 --decode > id_rsa.pub'
					
					//OCI CLI permissions mandatory on some files.
					sh 'oci setup repair-file-permissions --file ./bmcs_api_key.pem'
					
					sh 'ls'
					sh 'cat ./bmcs_api_key.pem'
					sh 'cat ./id_rsa'
					sh 'cat ./id_rsa.pub'
					
					env.TF_VAR_private_key_path = './bmcs_api_key.pem'
					echo "TF_VAR_private_key_path=${TF_VAR_private_key_path}"
					env.TF_VAR_ssh_private_key = sh returnStdout: true, script: 'cat ./id_rsa'
					echo "TF_VAR_ssh_private_key=${TF_VAR_ssh_private_key}"
					env.TF_VAR_ssh_public_key = sh returnStdout: true, script: 'cat ./id_rsa.pub'
					echo "TF_VAR_ssh_public_key=${TF_VAR_ssh_public_key}"
				}
				
				
				//OCI CLI Setup
				sh 'mkdir -p /root/.oci'
				sh 'rm -rf /root/.oci/config'
				sh 'echo "[DEFAULT]" > /root/.oci/config'
				sh 'echo "user=${TF_VAR_user_ocid}" >> /root/.oci/config'
				sh 'echo "fingerprint=${TF_VAR_fingerprint}" >> /root/.oci/config'
				sh 'echo "key_file=./bmcs_api_key.pem" >> /root/.oci/config'
				sh 'echo "tenancy=${TF_VAR_tenancy_ocid}" >> /root/.oci/config'
				sh 'echo "region=${TF_VAR_region}" >> /root/.oci/config'
				sh 'cat /root/.oci/config'
				
				//OCI CLI permissions mandatory on some files.
				sh 'oci setup repair-file-permissions --file /root/.oci/config'
            }
        }
		
		stage('Check Kubernetes Cluster') {
            steps {
				script {
					echo "CHOICE=${env.CHOICE}"
					
					//Check if Oke is already there
					sh 'oci ce cluster list --compartment-id=${TF_VAR_compartment_ocid} --name=Demo2_InfraAsCode_OKE --lifecycle-state=ACTIVE | jq ". | length" > result.test'	
					env.CHECK_OKE = sh (script: 'cat ./result.test', returnStdout: true).trim()
					sh 'echo ${CHECK_OKE}'
					
					if (env.CHECK_OKE == "1") {
						echo "Oke is running"
						env.OKE_CLUSTER_ID = sh returnStdout: true, script: 'oci ce cluster list --compartment-id=${TF_VAR_compartment_ocid} --name=Demo2_InfraAsCode_OKE --lifecycle-state=ACTIVE | jq -r .data[0].id'
						echo "OKE_CLUSTER_ID=${OKE_CLUSTER_ID}"
						
						//Get the kubeconfig file
						sh 'oci ce cluster create-kubeconfig --cluster-id=${OKE_CLUSTER_ID} --file=./kubeconfig'
						sh 'ls'
					}
					else {
						currentBuild.result = 'ABORTED'
						error('Oke not here or not running…')
					}	
				}	
            }
        }
		
		stage('Oracle Db') {
            steps {
				sh 'kubectl version'
			}
		}	
    }    
}
