
pipeline {
    agent any
 
    stages {
        stage('Import Flows') {
            steps {
                withCredentials([
                    string(credentialsId: 'boomi-flow-api-key', variable: 'FLOW_API_KEY')
                ]) {
                    script {
                        def config = readJSON file: 'config/config.json'
                        def tenantId = config.tenantId as String
                        def flowBaseUrl = config.flowBaseUrl as String
                        def overwriteExisting = params.OVERWRITE_EXISTING != null ? params.OVERWRITE_EXISTING.toBoolean() : (config.overwriteExisting ?: false)
                        def importTokens = params.IMPORT_TOKENS ? params.IMPORT_TOKENS.split('\n').collect { it.trim() }.findAll { it } : (config.importTokens ?: [])
 
                        if (!importTokens || importTokens.isEmpty()) {
                            echo "⚠️ No import tokens found in parameters or config. Skipping import stage."
                            return
                        }
 
                        echo "Found ${importTokens.size()} flow token(s) to import (overwriteExisting=${overwriteExisting})."
 
                        def commonHeaders = [
                            [name: 'manywhotenant', value: tenantId],
                            [name: 'x-boomi-flow-api-key', value: FLOW_API_KEY, maskValue: true]
                        ]
 
                        for (int i = 0; i < importTokens.size(); i++) {
                            def token = importTokens[i]
                            echo "Importing flow token [${i + 1}/${importTokens.size()}]: ${token.take(15)}..."
 
                            def payloadMap = [
                                token: token
                            ]
 
                            writeJSON file: 'import_flow_payload.json', json: payloadMap
                            def payloadJson = readFile file: 'import_flow_payload.json'
 
                            def response = httpRequest(
                                httpMode: 'POST',
                                ignoreSslErrors: true,
                                url: "${flowBaseUrl}/api/package/1/shared/flow?overwriteExisting=${overwriteExisting}",
                                contentType: 'APPLICATION_JSON',
                                requestBody: payloadJson,
                                customHeaders: commonHeaders,
                                validResponseCodes: '100:599',
                                consoleLogResponseBody: true
                            )
 
                            echo "HTTP Status: ${response.status}"
                            echo response.content
 
                            if (response.status >= 300) {
                                error("Flow import failed for token ${token.take(15)}... Status=${response.status}")
                            }
                            echo "✅ Successfully imported flow from token."
                        }
                        echo "✅ All flow imports completed successfully."
                    }
                }
            }
        }
 
        stage('Refresh Connectors') {
            steps {
                withCredentials([
                    string(credentialsId: 'boomi-flow-api-key', variable: 'FLOW_API_KEY')
                ]) {
                    script {
                        def config = readJSON file: 'config/config.json'
                        def connectors = config.connectors
 
                        if (!connectors || connectors.size() == 0) {
                            echo "⚠️ No connectors found in config. Skipping connector refresh stage."
                            return
                        }
 
                        echo "Found ${connectors.size()} connector(s) in configuration."
 
                        def commonHeaders = [
                            [name: 'manywhotenant', value: config.tenantId as String],
                            [name: 'x-boomi-flow-api-key', value: FLOW_API_KEY, maskValue: true]
                        ]
 
                        for (int i = 0; i < connectors.size(); i++) {
                            def connector = connectors[i]
                            echo "Installing / Refreshing connector: ${connector.developerName ?: connector.id} (${connector.id})"
 
                            def payloadMap = [
                                uri                                        : connector.uri as String,
                                developerName                              : connector.developerName as String,
                                developerSummary                           : connector.developerSummary ? connector.developerSummary as String : null,
                                httpAuthenticationUsername                 : connector.flowUsername as String,
                                httpAuthenticationPassword                 : connector.flowPassword as String,
                                HttpAuthenticationClientCertificate         : '',
                                HttpAuthenticationClientCertificatePassword: '',
                                configurationValues                        : [],
                                id                                         : connector.id as String,
                                identityProviderId                         : null
                            ]
 
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
                                error("Connector refresh failed for ${connector.id}. Status=${response.status}")
                            }
                            echo "✅ Successfully refreshed connector: ${connector.developerName ?: connector.id}"
                        }
                        echo "✅ All connector installations / refreshes completed successfully."
                    }
                }
            }
        }
 
        stage('Update Flow Identity Providers') {
            steps {
                withCredentials([
                    string(credentialsId: 'boomi-flow-api-key', variable: 'FLOW_API_KEY')
                ]) {
                    script {
                        def config = readJSON file: 'config/config.json'
                        def tenantId = config.tenantId as String
                        def flowBaseUrl = config.flowBaseUrl as String
                        def targetIdpId = params.IDP_ID ?: config.identityProviderId ?: config.idpId
                        def flowIds = params.FLOW_IDS ? params.FLOW_IDS.split('\n').collect { it.trim() }.findAll { it } : (config.flowIds ?: (config.masterFlowId ? [config.masterFlowId] : []))
 
                        if (!targetIdpId) {
                            echo "⚠️ No target Identity Provider ID specified in parameters or config. Skipping IDP update stage."
                            return
                        }
 
                        if (!flowIds || flowIds.isEmpty()) {
                            echo "⚠️ No Flow IDs found in parameters or config. Skipping IDP update stage."
                            return
                        }
 
                        echo "Target Identity Provider ID: ${targetIdpId}"
                        echo "Updating IDP for ${flowIds.size()} flow(s)..."
 
                        def commonHeaders = [
                            [name: 'manywhotenant', value: tenantId],
                            [name: 'x-boomi-flow-api-key', value: FLOW_API_KEY, maskValue: true]
                        ]
 
                        flowIds.each { flowId ->
                            echo "=== Fetching definition for Flow ID: ${flowId} ==="
 
                            // Step 1: GET flow definition
                            def getFlowResponse = httpRequest(
                                httpMode: 'GET',
                                ignoreSslErrors: true,
                                url: "${flowBaseUrl}/api/draw/1/flow/${flowId}",
                                customHeaders: commonHeaders,
                                validResponseCodes: '100:599',
                                consoleLogResponseBody: true
                            )
 
                            if (getFlowResponse.status >= 300) {
                                error("Failed to fetch Flow definition for Flow ID: ${flowId}. Status: ${getFlowResponse.status}")
                            }
 
                            def flowObj = readJSON(text: getFlowResponse.content)
 
                            // Step 2: Update identityProvider ID
                            if (!flowObj.identityProvider) {
                                flowObj.identityProvider = [:]
                            }
                            flowObj.identityProvider.id = targetIdpId
                            if (flowObj.identityProvider.allowedGroups == null) {
                                flowObj.identityProvider.allowedGroups = []
                            }
                            if (flowObj.identityProvider.allowedUsers == null) {
                                flowObj.identityProvider.allowedUsers = []
                            }
 
                            echo "Updating Flow ID: ${flowId} with Identity Provider ID: ${targetIdpId}"
 
                            writeJSON file: 'flow_update_payload.json', json: flowObj
                            def payloadJson = readFile file: 'flow_update_payload.json'
 
                            // Step 3: POST updated flow definition back to Flow Draw API
                            def saveResponse = httpRequest(
                                httpMode: 'POST',
                                ignoreSslErrors: true,
                                url: "${flowBaseUrl}/api/draw/1/flow",
                                contentType: 'APPLICATION_JSON',
                                requestBody: payloadJson,
                                customHeaders: commonHeaders,
                                validResponseCodes: '100:599',
                                consoleLogResponseBody: true
                            )
 
                            echo "HTTP Status: ${saveResponse.status}"
                            echo saveResponse.content
 
                            if (saveResponse.status >= 300) {
                                error("Failed to update IDP for Flow ID: ${flowId}. Status: ${saveResponse.status}")
                            }
 
                            echo "✅ Successfully updated Identity Provider for Flow ID: ${flowId}"
                        }
                        echo "✅ All Flow IDP updates completed successfully."
                    }
                }
            }
        }
 
        stage('Publish Master Flow') {
            steps {
                withCredentials([
                    string(credentialsId: 'boomi-flow-api-key', variable: 'FLOW_API_KEY')
                ]) {
                    script {
                        def config = readJSON file: 'config/config.json'
                        def tenantId = config.tenantId as String
                        def flowBaseUrl = config.flowBaseUrl as String
                        def masterFlowId = params.MASTER_FLOW_ID ?: config.masterFlowId ?: (config.flowIds ? config.flowIds[0] : null)
 
                        if (!masterFlowId) {
                            echo "⚠️ No Master Flow ID provided in build parameters or config. Skipping publish stage."
                            return
                        }
 
                        def commonHeaders = [
                            [name: 'manywhotenant', value: tenantId],
                            [name: 'x-boomi-flow-api-key', value: FLOW_API_KEY, maskValue: true]
                        ]
 
                        echo "=== Publishing Master Flow ID: ${masterFlowId} ==="
 
                        // Step 1: Look up the latest snapshot version for the master flow
                        def snapResponse = httpRequest(
                            httpMode: 'GET',
                            ignoreSslErrors: true,
                            url: "${flowBaseUrl}/api/draw/1/flow/snap/${masterFlowId}",
                            customHeaders: commonHeaders,
                            validResponseCodes: '100:599',
                            consoleLogResponseBody: true
                        )
 
                        if (snapResponse.status >= 300) {
                            error("Failed to list snapshots for Master Flow ID: ${masterFlowId}. Status: ${snapResponse.status}")
                        }
 
                        def snapshots = readJSON(text: snapResponse.content)
 
                        if (!(snapshots instanceof List) || snapshots.isEmpty()) {
                            error("No snapshots found for Master Flow ID: ${masterFlowId}")
                        }
 
                        // Sort explicitly by dateCreated
                        def latestSnapshot = snapshots.max { it.dateCreated }
                        def versionId = latestSnapshot.id.versionId
 
                        if (!versionId) {
                            error("Could not determine version ID for Master Flow ID: ${masterFlowId}")
                        }
 
                        echo "Latest version for Master Flow ID ${masterFlowId}: ${versionId} (created ${latestSnapshot.dateCreated})"
 
                        // Step 2: Activate that version and make it the default
                        def activateResponse = httpRequest(
                            httpMode: 'POST',
                            ignoreSslErrors: true,
                            url: "${flowBaseUrl}/api/draw/1/flow/activation/${masterFlowId}/${versionId}/true/true",
                            customHeaders: commonHeaders,
                            validResponseCodes: '100:599',
                            consoleLogResponseBody: true
                        )
 
                        echo "Activation status: ${activateResponse.status}"
                        echo "Activation body: ${activateResponse.content}"
 
                        if (activateResponse.status >= 300) {
                            error("Publish failed for Master Flow ID: ${masterFlowId}. Status=${activateResponse.status}")
                        } else {
                            echo "✅ Master Flow ID ${masterFlowId} published successfully (version ${versionId})"
                            echo "🚀 Flow Launch URL: ${flowBaseUrl}/${tenantId}/play/theme/default/?flow-id=${masterFlowId}"
                        }
                    }
                }
            }
        }
    }
}
 
 