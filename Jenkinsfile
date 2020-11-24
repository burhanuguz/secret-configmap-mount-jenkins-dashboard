def apiRequest(String apiReq, String method) {
	if ( oauthToken == "" ) {
		withCredentials([string(credentialsId: "${params.credID}", variable: 'TOKEN')]) {
			sh (
				script: "curl ${insecure} -X ${method} -H \"Authorization: Bearer ${TOKEN}\" ${apiReq}",
				returnStdout: true
			)
		}
	}
	else {
		sh ( script: "curl ${insecure} -X ${method} -H \"Authorization: Bearer ${oauthToken}\" ${apiReq}", returnStdout: true )
	}
}
pipeline {
	agent any
	parameters {
		string       ( name: 'credID'        , description: 'Give Credentials ID to authorize Cluster(User-Pass or TOKEN)'                                      , defaultValue: 'apiToken'                                                                                                                                                                                                                                                                    )
		string       ( name: 'apiServer'     , description: 'API Server of The Cluster'                                                                         , defaultValue: 'https://IP:PORT'                                                                                                                                                                                                                                                             )
		booleanParam ( name: 'Insecure'      , description: 'Insecure flag'                                                                                     , defaultValue: false                                                                                                                                                                                                                                                                         )
		booleanParam ( name: 'Openshift'     , description: 'Is it Openshift Cluster?'                                                                          , defaultValue: false                                                                                                                                                                                                                                                                         )
		booleanParam ( name: 'OpenshiftUser' , description: 'Use Username and Password to Authenticate? (Openshift Only)'                                       , defaultValue: false                                                                                                                                                                                                                                                                         )
		string       ( name: 'namespace'     , description: 'You can manually type Namespace/Project. Leave it empty if you want to choose from namespace list' , defaultValue: '{namespace/project}'                                                                                                                                                                                                                                                         )
		choice       ( name: 'resourceType'  , description: 'Select Resource Type to Mount'                                                                     , choices: ['Deployment','Statefulset','Daemonset']                                                                                                                                                                                                                                           )
		string       ( name: 'resourceName'  , description: 'You can manually type Workload Name. If nothing is typed you will choose combobox'                 , defaultValue: '{workloadName}'                                                                                                                                                                                                                                                              )
		choice       ( name: 'mountType'     , description: 'Select Mount Type'                                                                                 , choices: ['Secret Volume Mount','ConfigMap Volume Mount','Define all of the Secret data as Environment Variable','Define all of the ConfigMap data as Environment Variable','Define specific Secret data as Environment Variable','Define specific ConfigMap data as Environment Variable'] )
		string       ( name: 'mountNumber'   , description: 'Number of Secret or Config Map to create. (1 < Number < 20)'                                       , defaultValue: '1'                                                                                                                                                                                                                                                                           )
	}
	stages {
		stage('Selecting Project and Resource Type') {
			steps {
				script {
					oauthToken = ""
					if ( "${params.mountNumber}" < 1 || "${params.mountNumber}" > 20 ) { error "Limit exceed for creating Secret/ConfigMap" }
					resourceType = "${params.resourceType.toLowerCase().replaceAll(' ','')}" + "s"
					if ( params.Insecure  == true ) { insecure = "-k" }
					else                            { insecure =  ""  }
					if ( params.Openshift == true ) {
						request  = "apis/project.openshift.io/v1/projects"
						if ( params.OpenshiftUser == true ) {
							oauthUrl = readJSON text: sh( script: "curl ${params.apiServer}/.well-known/oauth-authorization-server", returnStdout: true )
							withCredentials([usernamePassword(credentialsId: "${params.credID}", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD', TOKEN: '')]) {
								oauthToken = sh ( script: "sh -c \'curl -u \"${USERNAME}:${PASSWORD}\" --head -H \"X-CSRF-Token: 1\" \"${oauthUrl.issuer}/oauth/authorize?client_id=openshift-challenging-client&response_type=token\" | grep -o \"token=.*&\" | cut -d \"&\" -f 1 | cut -d \"=\" -f 2\'", returnStdout: true ).trim()
							}
						}
					}
					else {
						request  = "api/v1/namespaces"
					}
					namespaces = readJSON text: apiRequest("\"${params.apiServer}/${request}\"", "GET")
					if ( "${params.namespace}" == "" ) {
						namespace = input (
							message: "Select Namespace/Project",
							ok: "OK",
							parameters: [ choice (name: 'namespace', choices: namespaces.items.metadata.name) ]
						)
					}
					else {
						assert namespaces.items.metadata.name.find { it == "${params.namespace}" }
						namespace = "${params.namespace}"
					}
					resourceList = readJSON text: apiRequest("\"${params.apiServer}/apis/apps/v1/namespaces/${namespace}/${resourceType}\"", "GET")
					if ( "${params.resourceName}" == "" ) {
						if ( resourceList.items.metadata.name.size() != 0 ) {
							resourceName = input (
								message: "Select Resource",
								ok: "OK",
								parameters: [ choice (name: 'resourceName', choices: resourceList.items.metadata.name) ]
							)
						}
					}
					else {
						assert resourceList.items.metadata.name.find { it == "${params.resourceName}" }
						resourceName = "${params.resourceName}"
					}
					containerList = readJSON text: apiRequest("\"${params.apiServer}/apis/apps/v1/namespaces/${namespace}/${resourceType}/${resourceName}\"", "GET")
					if ( params.mountType == "Secret Volume Mount" || params.mountType == "Define specific Secret data as Environment Variable" || params.mountType == "Define all of the Secret data as Environment Variable"  ) {
						mountTypeLocal = "Secret"
						dataPref       = "secretName"
						mountType      = "secret"
					}
					else if ( params.mountType == "ConfigMap Volume Mount" || params.mountType == "Define specific ConfigMap data as Environment Variable" || params.mountType == "Define all of the ConfigMap data as Environment Variable" ) {
						mountTypeLocal = "ConfigMap"
						dataPref       = "name"
						mountType      = "configMap"
					}
					else { 
						error "Error"
					}
				}
			}
		}
		stage('Create Stage') {
			steps {
				script {
					def paramchoice = []
					int x = 0
					for ( int i = 0; i < (params.mountNumber as Integer)*2; i = i + 2 ) {
						paramchoice [  i  ] = string ( name: "mountName-${i/2}" , defaultValue: "example-${i/2}" , description: "kind: ${mountTypeLocal}\nmetadata:\n  namespace: ${namespace}\n  name:" )
						paramchoice [i + 1] = string ( name: "numberOf-${i/2}"  , defaultValue: "1"              , description: "Number of File/Variable/Data to be added in ${mountTypeLocal}-${i/2}"   )
					}
					mountInfo = input (
						message: "Enter ${mountTypeLocal} Information",
						ok: "OK",
						parameters: paramchoice
					)
					paramchoice = []
					for ( int i = 0; i < (params.mountNumber as Integer); i++ ) {
						for (int j = 0; j < (mountInfo.("numberOf-" + "${i}") as Integer); j++ ) {
							paramchoice [  x  ] = string ( name: "keyName-${x/2}" , defaultValue: "key-${x/2}"   , description: "kind: ${mountTypeLocal}\nmetadata:\n  namespace: ${namespace}\n  name: ${mountInfo.("mountName-" + "${i}")}\ndata:" )
							paramchoice [x + 1] = text   ( name: "keyInfo-${x/2}" , defaultValue: "value-${x/2}" , description: "key-${x/2}'s Value-${x/2}" )
							x = x + 2
						}
					}
					x = 0
					plainMounts = input (
						message: "Enter Values of ${mountTypeLocal}",
						ok: "OK",
						parameters: paramchoice
					)
					paramchoice = []
					for ( int i = 0; i < (params.mountNumber as Integer); i++ ) {
						tempResult = []
						for (int j = 0; j < (mountInfo.("numberOf-" + "${i}") as Integer); j++ ) {
							if      ( params.mountType == "Secret Volume Mount"    || params.mountType == "Define specific Secret data as Environment Variable"    || params.mountType == "Define all of the Secret data as Environment Variable"    ) { keyInfo = sh ( script:"echo \'${plainMounts.("keyInfo-" + "${x}")}\' | base64 | tr -d '\n'"                  , returnStdout: true ).trim() }
							else if ( params.mountType == "ConfigMap Volume Mount" || params.mountType == "Define specific ConfigMap data as Environment Variable" || params.mountType == "Define all of the ConfigMap data as Environment Variable" ) { keyInfo = sh ( script:"echo \'${plainMounts.("keyInfo-" + "${x}")}\' | sed \':a;N;\$!ba;s/\\n/\\\\\\\\n/g\'" , returnStdout: true ).trim() }
							tempResult [j] = "\\\"" + plainMounts.("keyName-" + "${x}") + "\\\":" + "\\\"" + keyInfo + "\\\""
							x++
						}
						data = sh ( script: "echo {\\\"apiVersion\\\": \\\"v1\\\",\\\"data\\\": {" + tempResult + "}, \\\"kind\\\": \\\"" + mountTypeLocal + "\\\",\\\"metadata\\\": {\\\"name\\\": \\\"" + mountInfo.("mountName-" + "${i}") + "\\\"}}", returnStdout: true ).trim().replaceAll('\\]','').replaceAll('\\[','')
						apiRequest("-H \"Content-Type: application/json\" \"${params.apiServer}/api/v1/namespaces/${namespace}/${mountType}s\" --data \'${data}\'", "POST")
					}
				}
			}
		}
		stage('Mount Stage') {
			steps {
				script {
					def paramchoice = []
					def data        = []
					def result      = []
					def limit       = []
					int x = 0
					int y = 0
					mountTypeList = readJSON text: apiRequest("\"${params.apiServer}/api/v1/namespaces/${namespace}/${mountType}s\"", "GET")
					if ( params.mountType == "Secret Volume Mount" || params.mountType == "ConfigMap Volume Mount" ) {
						def volumeList      = []
						def indexOf         = []
						def isSpecific      = []
						def tempItemResult  = []
						def volMountNumber  = []
						def tempResult      = []
						def containerMounts = []
						def volumeMount     = []
						for ( int i = 0; i < (params.mountNumber as Integer)*2; i = i + 2 ) {
							paramchoice [  i  ] = string       ( name: "volName_${i/2}"      , defaultValue: "config-volume-${i/2}" , description: "volumes:\n  - name:  ${mountType}: \n    ${dataPref}: ${mountInfo.("mountName-" + "${i/2}")}" )
							paramchoice [i + 1] = booleanParam ( name: "${i/2}-Is Specific?" , defaultValue: false                  , description: "If you want to mount ${mountTypeLocal}'s data to a specific path in the Volume click it"      )
						}
						volMounts = input (
							message: "Decide Volumes Names. \nExample:\n \
							volumes:\n  - name: vol-name\n    ${mountType}:\n       ${dataPref}: ${mountType}-name\n       ## Is Specific Part ##\n       items:\n       - key: SPECIAL_LEVEL\n         path: keys",
							ok: "OK",
							parameters: paramchoice
						)
						paramchoice = []
						for ( int i = 0; i < (params.mountNumber as Integer); i++ ) {
							volumeList [i] = volMounts.("volName_" + "${i}")
							if ( volMounts.("${i}-" +"Is Specific?" ) == true ) {
								paramchoice [x] = string ( name: "${x}" , defaultValue: "1" , description: mountInfo.("mountName-" + "${i}") )
								indexOf     [x] = mountTypeList.items.metadata.name.findIndexOf { it == mountInfo.("mountName-" + "${i}") }
								isSpecific  [x] = volMounts.("volName_" + "${i}")
								tempResult  [x] = "{\\\"name\\\": \\\"" + volMounts.("volName_" + "${i}") + "\\\", \\\"" + mountType + "\\\": {\\\"" + dataPref + "\\\": \\\"" + mountInfo.("mountName-" + "${i}") + "\\\", \\\"items\\\": "
								x++
							}
							else {
								result [i] = "{\\\"name\\\": \\\"" + volMounts.("volName_" + "${i}") + "\\\", \\\"" + mountType + "\\\": {\\\"" + dataPref + "\\\": \\\"" + mountInfo.("mountName-" + "${i}") + "\\\"}}"
							}
						}
						if ( x != 0 ) {
							specificVolumeInput = input (
								message: "Select how many key and path to add specifically for the ${mountType}",
								ok: "OK",
								parameters: paramchoice
							)
							paramchoice = []
							for ( int i = 0 ; i < specificVolumeInput.size(); i++ ) {
								if ( x == 1 ) { limit [i] = (specificVolumeInput          as Integer) }
								else          { limit [i] = (specificVolumeInput.("${i}") as Integer) }
								for ( int j = 0 ; j < limit [i]; j++ ) {
									paramchoice [  y  ] = choice ( name: "key_${y/2}"  , choices: (mountTypeList.items[indexOf [i]].data.collect { "${it}" }).collect { it.replaceAll('\n.*','').replaceAll('=.*','') } , description: "volumes:\n  - name: ${isSpecific [i]}\n    ${mountType}:\n      ${dataPref}: ${mountTypeList.items[indexOf [i]].metadata.name}\n      items:\n      - key: " )
									paramchoice [y + 1] = string ( name: "path_${y/2}" , defaultValue: "keys_${y/2}", description: "         path:" )
									y = y + 2
								}
							}
							keyAndPathInput = input (
								message: "Select which key to mount and then type path",
								ok: "OK",
								parameters: paramchoice
							)
							paramchoice = []
							x = 0
							for ( int i = 0 ; i < specificVolumeInput.size(); i++ ) {
								for ( int j = 0 ; j < limit[i]; j++ ) {
									tempItemResult [j] = "{\\\"key\\\": \\\"" + keyAndPathInput.("key_" + "${x}") + "\\\", \\\"path\\\": \\\"" + keyAndPathInput.("path_" + "${x}") + "\\\"}"
									x++
								}
								tempResult [i] = tempResult [i] + tempItemResult + "}}"
							}
						}
						x = 0
						for ( int i = 0; i <= (params.mountNumber as Integer); i++ ) {
							volMountNumber [i] = i
							if ( volMounts.("${i}-" +"Is Specific?" ) == true ) {
								result [i] = tempResult [x++]
							}
						}
						for ( int i = 0; i < result.size(); i++ ) {
							data = sh ( script: "echo {\\\"spec\\\":{\\\"template\\\":{\\\"spec\\\":{\\\"volumes\\\": [" + "${result[i]}" + "]}}}}", returnStdout: true ).trim()
							apiRequest("-H \"Content-Type: application/strategic-merge-patch+json\" \"${params.apiServer}/apis/apps/v1/namespaces/${namespace}/${resourceType}/${resourceName}\" --data \'${data}\'", "PATCH")
						}
						for ( int i = 0; i <  containerList.spec.template.spec.containers.name.size(); i++ ) {
							paramchoice [i] = choice ( name: "volMountNumber_${i}", choices: volMountNumber, description: containerList.spec.template.spec.containers[i].name )
						}
						totalVolMount = input (
							message: "Select how many volume to mount on the specified Container",
							ok: "OK",
							parameters: paramchoice
						)
						limit = []
						paramchoice = []
						x = 0
						for ( int i = 0; i < containerList.spec.template.spec.containers.name.size(); i++ ) {
							if ( containerList.spec.template.spec.containers.name.size() == 1 ) { limit [i] = totalVolMount }
							else { limit [i] = totalVolMount.("volMountNumber_" + "${i}") }
							for ( int j = 0; j < (limit [i] as Integer); j++ ) {
								paramchoice [  x  ] = choice ( name: "mountName_${x/2}" , choices: volumeList, description: "- containers:\n  name: ${containerList.spec.template.spec.containers[i].name}\n  volumeMounts:\n    - name:" )
								paramchoice [x + 1] = string ( name: "mountPath_${x/2}" , defaultValue: "/tmp-${x/2}", description: "  mountPath:" )
								x = x + 2
							}
						}
						mountVol = input (
							message: "Now select volumes mount Container to mount the volume. \
							Then type Volume Mount Name and mountPath for the volume and select ${mountTypeLocal}",
							ok: "OK",
							parameters: paramchoice
						)
						paramchoice = []
						for ( int i = 0; i < containerList.spec.template.spec.containers.name.size(); i++ ) {
							x = 0
							volumeMount = []
							for ( int j = 0; j < (limit [i] as Integer); j++ ) {
								volumeMount [x] = "{\\\"name\\\": \\\"" + mountVol.("mountName_" + "${x}") + "\\\", \\\"mountPath\\\": \\\"" + mountVol.("mountPath_" + "${x}") + "\\\"}"
								x++
							}
							containerMounts [i] = "{\\\"name\\\": \\\"" + containerList.spec.template.spec.containers[i].name + "\\\", \\\"volumeMounts\\\": " + volumeMount + "}"
						}
						data = sh ( script: "echo {\\\"spec\\\":{\\\"template\\\":{\\\"spec\\\":{\\\"containers\\\": " + "${containerMounts}" + "}}}}", returnStdout: true ).trim()
						apiRequest("-H \"Content-Type: application/strategic-merge-patch+json\" \"${params.apiServer}/apis/apps/v1/namespaces/${namespace}/${resourceType}/${resourceName}\" --data \'${data}\'", "PATCH")
					}
					else if ( params.mountType == "Define specific Secret data as Environment Variable" || params.mountType == "Define specific ConfigMap data as Environment Variable" ) {
						def tempKeyRefs   = []
						def tempRefResult = []
						for ( int i = 0; i < (params.mountNumber as Integer); i++ ) {
							paramchoice [i] = string ( name: "numberOfKey-${i}" , defaultValue: "1" , description: "Number of Variable/Data to get from ${mountInfo.("mountName-" + "${i}")} as Environment Variable" ) 
						}
						mountList = input (
							message: "Select how many key to mount from ${mountTypeLocal}'s\n \
							Example Referenced Value: \n\nenv:\n- name: SPECIAL_LEVEL_KEY\n  valueFrom:\n    ${mountType}KeyRef:\n      name: ${mountType}-name\n      key: SPECIAL_LEVEL",
							ok: "OK",
							parameters: paramchoice
						)
						paramchoice = []
						for ( int i = 0; i < (params.mountNumber as Integer); i++ ) {
							if ( mountList.size() != 1 ) { limit [i] = (mountList.("numberOfKey-" + "${i}") as Integer) }
							else                         { limit [i] = (mountList as Integer)                           }
							for (int j = 0; j < limit[i]; j++ ) {
								paramchoice [  x  ] = string ( name: "keyInfo_${x/2}" , defaultValue: "SPECIAL_LEVEL_KEY-${x/2}" , description: "env:\n- name:" )
								paramchoice [x + 1] = choice ( name: "keyName_${x/2}" , choices: (mountTypeList.items[ mountTypeList.items.metadata.name.findIndexOf { it == mountInfo.("mountName-" + "${i}") } ].data.collect { "${it}" }).collect { it.replaceAll('\n.*','').replaceAll('=.*','') }, description: "  valueFrom:\n    ${mountType}KeyRef:\n      name: ${mountInfo.("mountName-" + "${i}")}\n      key:" )
								x = x + 2
							}
						}
						refMounts = input (
							message: "Enter Values of ${mountTypeLocal} values.",
							ok: "OK",
							parameters: paramchoice
						)
						paramchoice = []
						x = 0
						for ( int i = 0; i < (params.mountNumber as Integer); i++ ) {
							for (int j = 0; j < limit[i]; j++ ) {
								tempKeyRefs [x] = "{\\\"name\\\": \\\"" + refMounts.("keyInfo_" + "${x}") + "\\\", \\\"valueFrom\\\": {\\\"" + mountType + "KeyRef\\\": {\\\"name\\\": \\\"" + mountInfo.("mountName-" + "${i}") + "\\\",\\\"key\\\": \\\"" + refMounts.("keyName_" + "${x}") + "\\\"}}}"
								x++
							}
						}
						for ( int i = 0; i < containerList.spec.template.spec.containers.name.size(); i++ ) {
							x = 0
							for ( int j = 0; j < (params.mountNumber as Integer); j++ ) {
								for ( int k = 0; k < limit[j]; k++ ) {
									paramchoice [y] = booleanParam ( name: "${containerList.spec.template.spec.containers[i].name} ${refMounts.("keyInfo_" + "${x}")}" , defaultValue: false , description: "Mount ${refMounts.("keyInfo_" + "${x}")} to ${containerList.spec.template.spec.containers[i].name}?" )
									x++
									y++
								}
							}
						}
						containerRefMount = input (
							message: "To which containers mount the environment variables?",
							ok: "OK",
							parameters: paramchoice
						)
						paramchoice = []
						for ( int i = 0; i < containerList.spec.template.spec.containers.name.size(); i++ ) {
							if ( containerList.spec.template.spec.containers.name.size() == 1 && (params.mountNumber as Integer) == 1 && (mountList as Integer) == 1 && containerRefMount == true ) {
								result = "{\\\"name\\\": \\\"" + containerList.spec.template.spec.containers[i].name + "\\\", \\\"env\\\": " + tempKeyRefs + "}"
							}
							else {
								tempRefResult = []
								x = 0
								for ( int j = 0; j < (params.mountNumber as Integer); j++ ) {
									for ( int k = 0; k < limit [j]; k++ ) {
										if ( containerRefMount.( "${containerList.spec.template.spec.containers[i].name} ${refMounts.("keyInfo_" + "${k}")}" ) == true ) {
											tempRefResult.add( tempKeyRefs [x] )
										}
										x++
									}
								}
								if ( tempRefResult.size() != 0 ) { 
									result [i] = "{\\\"name\\\": \\\"" + containerList.spec.template.spec.containers[i].name + "\\\", \\\"env\\\": " + tempRefResult + "}"
								}
							}
							if ( result != "" ) { 
								data = sh ( script: "echo {\\\"spec\\\":{\\\"template\\\":{\\\"spec\\\":{\\\"containers\\\": " + "${result}" + "}}}}", returnStdout: true ).trim()
								apiRequest("-H \"Content-Type: application/strategic-merge-patch+json\" \"${params.apiServer}/apis/apps/v1/namespaces/${namespace}/${resourceType}/${resourceName}\" --data \'${data}\'", "PATCH")
							}
						}
					}
					else if ( params.mountType == "Define all of the Secret data as Environment Variable" || params.mountType == "Define all of the ConfigMap data as Environment Variable" ) {
						def tempEnvRefs    = []
						def tempDirResult  = []
						if ( (params.mountNumber as Integer) != 0 ) {
							for ( int i = 0; i < (params.mountNumber as Integer); i++ ) {
								tempEnvRefs [i] = "{\\\"" + mountType + "Ref\\\":{ \\\"name\\\": \\\"" + mountInfo.("mountName-" + "${i}") + "\\\"}}"
							}
							x = 0
							for ( int i = 0; i < containerList.spec.template.spec.containers.name.size(); i++ ) {
								for ( int j = 0; j < (params.mountNumber as Integer); j++ ) {
									paramchoice [x] = booleanParam ( name: "${containerList.spec.template.spec.containers[i].name} ${mountInfo.("mountName-" + "${i}")}" , defaultValue: false , description: "Mount ${mountInfo.("mountName-" + "${i}")} to ${containerList.spec.template.spec.containers[i].name}?" )
									x++
								}
							}
							containerDirMount = input (
								message: "To which containers mount the environment variables?",
								ok: "OK",
								parameters: paramchoice
							)
							paramchoice = []
							if ( containerList.spec.template.spec.containers.name.size() == 1 && (params.mountNumber as Integer) == 1 && containerDirMount == true ) {
								result = "{\\\"name\\\": \\\"" + containerList.spec.template.spec.containers[0].name + "\\\" , \\\"envFrom\\\": " + tempEnvRefs + "}"
							}
							else {
								for ( int i = 0; i < containerList.spec.template.spec.containers.name.size(); i++ ) {
									tempDirResult = []
									for ( int j = 0; j < (params.mountNumber as Integer); j++ ) {
										if ( containerDirMount.( "${containerList.spec.template.spec.containers[i].name} ${mountInfo.("mountName-" + "${i}")}" ) == true ) {
											tempDirResult.add( tempEnvRefs [j] )
										}
									}
									if (tempDirResult.size() != 0 ) {
										result [i] = "{\\\"name\\\": \\\"" + containerList.spec.template.spec.containers[i].name + "\\\" , \\\"envFrom\\\": " + tempDirResult + "}"
									}
								}
							}
							if ( result != "" ) {
								data = sh ( script: "echo {\\\"spec\\\":{\\\"template\\\":{\\\"spec\\\":{\\\"containers\\\": " + "${result}" + "}}}}", returnStdout: true ).trim()
								apiRequest("-H \"Content-Type: application/strategic-merge-patch+json\" \"${params.apiServer}/apis/apps/v1/namespaces/${namespace}/${resourceType}/${resourceName}\" --data \'${data}\'", "PATCH")
							}
						}
					}
				}
			}
		}
	}
	post {
		always {
			script {
				if ( oauthToken != "" ) {
					sh ( script: "curl -k -X DELETE -H \"Authorization: Bearer ${oauthToken}\" \'${params.apiServer}/apis/oauth.openshift.io/v1/oauthaccesstokens/${oauthToken}\'" )
				}
			}
		}
	}
}
