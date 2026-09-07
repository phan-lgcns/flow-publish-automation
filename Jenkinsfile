pipeline {
    agent any

    stages {
        stage('Install Connectors') {
            steps {
                withCredentials([
                    string(credentialsId: 'boomi-flow-api-key', variable: 'FLOW_API_KEY')
                ]) {
                    script {
                        def config = readJSON file: 'config/config.json'
                        def connectors = config.connectors

                        if (!connectors || connectors.size() == 0) {
                            echo "⚠️ No connectors found in config. Skipping connector install stage."
                            return
                        }

                        echo "Found ${connectors.size()} connector(s) in configuration."

                        def commonHeaders = [
                            [name: 'manywhotenant', value: config.tenantId as String],
                            [name: 'x-boomi-flow-api-key', value: FLOW_API_KEY, maskValue: true]
                        ]

                        for (int i = 0; i < connectors.size(); i++) {
                            def connector = connectors[i]
                            echo "Installing connector: ${connector.id}"

                            // Build the payload map
                            def payloadMap = [
                                uri                                  : connector.uri as String,
                                developerName                        : connector.developerName as String,
                                developerSummary                     : connector.developerSummary ? connector.developerSummary as String : null,
                                httpAuthenticationUsername            : config.flowUsername as String,
                                httpAuthenticationPassword           : config.flowPassword as String,
                                HttpAuthenticationClientCertificate   : '',
                                HttpAuthenticationClientCertificatePassword: '',
                                configurationValues                  : [],
                                id                                   : connector.id as String,
                                identityProviderId                   : null
                            ]

                            // Serialize using pipeline-safe writeJSON
                            writeJSON file: 'connector_payload.json', json: payloadMap
                            def payloadJson = readFile file: 'connector_payload.json'

                            def response = httpRequest(
                                httpMode: 'POST',
                                ignoreSslErrors: true,
                                url: "${config.flowBaseUrl}/api/draw/1/element/service/install",
                                contentType: 'APPLICATION_JSON',
                                requestBody: payloadJson,
                                customHeaders: commonHeaders,
                                validResponseCodes: '100:599',
                                consoleLogResponseBody: true
                            )

                            echo "HTTP Status: ${response.status}"
                            echo response.content

                            if (response.status >= 300) {
                                error("Connector installation failed for ${connector.id}. Status=${response.status}")
                            }
                            echo "✅ Successfully installed connector ${connector.id}"
                        }
                        echo "✅ All connector installations completed successfully"
                    }
                }
            }
        }

        stage('Configure Identity Providers') {
            steps {
                withCredentials([
                    string(credentialsId: 'boomi-flow-api-key', variable: 'FLOW_API_KEY')
                ]) {
                    script {
                        def config = readJSON file: 'config/config.json'
                        def identityProviders = config.identityProviders

                        if (!identityProviders || identityProviders.size() == 0) {
                            echo "⚠️ No identity providers found in config. Skipping identity provider stage."
                            return
                        }

                        echo "Found ${identityProviders.size()} identity provider(s) in configuration."

                        def commonHeaders = [
                            [name: 'manywhotenant', value: config.tenantId as String],
                            [name: 'x-boomi-flow-api-key', value: FLOW_API_KEY, maskValue: true],
                            [name: 'Content-Type', value: 'application/json']
                        ]

                        for (int i = 0; i < identityProviders.size(); i++) {
                            def idp = identityProviders[i]
                            echo "Configuring identity provider: ${idp.developerName} (${idp.id})"

                            // Build the payload — include all fields from the identity provider config
                            def payloadMap = [
                                '$type'                     : idp.'$type' as String,
                                attributeMappings           : [
                                    role     : idp.attributeMappings?.role,
                                    email    : idp.attributeMappings?.email,
                                    groups   : idp.attributeMappings?.groups,
                                    lastname : idp.attributeMappings?.lastname,
                                    firstname: idp.attributeMappings?.firstname
                                ],
                                wellKnownUrl                : idp.wellKnownUrl as String,
                                clientId                    : idp.clientId as String,
                                clientSecret                : idp.clientSecret,
                                allowedAudience             : idp.allowedAudience ? idp.allowedAudience as String : null,
                                resource                    : idp.resource ? idp.resource as String : '',
                                scope                       : idp.scope as String,
                                sendAccessTokenToConnectors : idp.sendAccessTokenToConnectors as Boolean,
                                grantTypes                  : idp.grantTypes,
                                pkce                        : idp.pkce as String,
                                type                        : idp.'$type' as String,
                                enhancedTokenSecurity       : idp.enhancedTokenSecurity as Boolean,
                                id                          : idp.id as String,
                                elementType                 : 'IDENTITY_PROVIDER',
                                developerName               : idp.developerName as String,
                                developerSummary            : idp.developerSummary ? idp.developerSummary as String : ''
                            ]

                            // Serialize using pipeline-safe writeJSON
                            writeJSON file: 'idp_payload.json', json: payloadMap
                            def payloadJson = readFile file: 'idp_payload.json'

                            def response = httpRequest(
                                httpMode: 'POST',
                                ignoreSslErrors: true,
                                url: "${config.flowBaseUrl}/api/draw/1/element/identityprovider",
                                contentType: 'APPLICATION_JSON',
                                requestBody: payloadJson,
                                customHeaders: commonHeaders,
                                validResponseCodes: '100:599',
                                consoleLogResponseBody: true
                            )

                            echo "HTTP Status: ${response.status}"
                            echo response.content

                            if (response.status >= 300) {
                                error("Identity provider configuration failed for ${idp.developerName}. Status=${response.status}")
                            }
                            echo "✅ Successfully configured identity provider: ${idp.developerName}"
                        }
                        echo "✅ All identity provider configurations completed successfully"
                    }
                }
            }
        }

        stage('Publish Flows') {
            steps {
                withCredentials([
                    string(credentialsId: 'boomi-flow-api-key', variable: 'FLOW_API_KEY')
                ]) {
                    script {
                        def config = readJSON file: 'config/config.json'
                        def tenantId = config.tenantId as String
                        def flowBaseUrl = config.flowBaseUrl as String
                        def flowIds = params.FLOW_IDS ? params.FLOW_IDS.split('\n').collect { it.trim() }.findAll { it } : config.flowIds

                        if (!flowIds || flowIds.isEmpty()) {
                            echo "⚠️ No Flow IDs provided in build parameters or config. Skipping flow publish stage."
                            return
                        }

                        def commonHeaders = [
                            [name: 'manywhotenant', value: tenantId],
                            [name: 'x-boomi-flow-api-key', value: FLOW_API_KEY, maskValue: true]
                        ]

                        flowIds.each { flowId ->
                            echo "=== Processing Flow ID: ${flowId} ==="

                            // Step 1: Look up the latest snapshot version for this flow
                            def snapResponse = httpRequest(
                                httpMode: 'GET',
                                url: "${flowBaseUrl}/api/draw/1/flow/snap/${flowId}",
                                customHeaders: commonHeaders,
                                validResponseCodes: '100:599',
                                consoleLogResponseBody: true
                            )

                            if (snapResponse.status >= 300) {
                                error("Failed to list snapshots for Flow ID: ${flowId}. Status: ${snapResponse.status}")
                            }

                            def snapshots = readJSON(text: snapResponse.content)

                            if (!(snapshots instanceof List) || snapshots.isEmpty()) {
                                error("No snapshots found for Flow ID: ${flowId}")
                            }

                            // Sort explicitly by dateCreated
                            def latestSnapshot = snapshots.max { it.dateCreated }
                            def versionId = latestSnapshot.id.versionId

                            if (!versionId) {
                                error("Could not determine version ID for Flow ID: ${flowId}")
                            }

                            echo "Latest version for Flow ID ${flowId}: ${versionId} (created ${latestSnapshot.dateCreated})"

                            // Step 2: Activate that version and make it the default
                            def activateResponse = httpRequest(
                                httpMode: 'POST',
                                url: "${flowBaseUrl}/api/draw/1/flow/activation/${flowId}/${versionId}/true/true",
                                customHeaders: commonHeaders,
                                validResponseCodes: '100:599',
                                consoleLogResponseBody: true
                            )

                            echo "Activation status: ${activateResponse.status}"
                            echo "Activation body: ${activateResponse.content}"

                            if (activateResponse.status >= 300) {
                                error("Publish failed for Flow ID: ${flowId}")
                            } else {
                                echo "✅ Flow ID ${flowId} published successfully (version ${versionId})"
                                echo "${flowBaseUrl}/${tenantId}/play/theme/default/?flow-id=${flowId}"
                            }
                        }
                    }
                }
            }
        }
    }
}
