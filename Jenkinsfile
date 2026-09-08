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
                        def initialFlowIds = params.FLOW_IDS ? params.FLOW_IDS.split('\n').collect { it.trim() }.findAll { it } : (config.flowIds ?: (config.masterFlowId ? [config.masterFlowId] : []))

                        if (!targetIdpId) {
                            echo "⚠️ No target Identity Provider ID specified in parameters or config. Skipping IDP update stage."
                            return
                        }

                        if (!initialFlowIds || initialFlowIds.isEmpty()) {
                            echo "⚠️ No Flow IDs found in parameters or config. Skipping IDP update stage."
                            return
                        }

                        def commonHeaders = [
                            [name: 'manywhotenant', value: tenantId],
                            [name: 'x-boomi-flow-api-key', value: FLOW_API_KEY, maskValue: true]
                        ]

                        // Step 1: Discover all subflows from graph with JSONNull-safe parsing
                        def queue = [] as List
                        queue.addAll(initialFlowIds)
                        def visitedFlows = [] as Set
                        visitedFlows.addAll(initialFlowIds)
                        def discoveredSubflows = [] as List

                        echo "Discovering subflows from initial flow(s): ${initialFlowIds}..."

                        while (!queue.isEmpty()) {
                            def currentFlowId = queue.remove(0)
                            def graphResponse = httpRequest(
                                httpMode: 'GET',
                                ignoreSslErrors: true,
                                url: "${flowBaseUrl}/api/draw/2/graph/flow/${currentFlowId}",
                                customHeaders: commonHeaders,
                                validResponseCodes: '100:599',
                                consoleLogResponseBody: false
                            )

                            if (graphResponse.status < 300 && graphResponse.content) {
                                def graphObj = readJSON(text: graphResponse.content)
                                if (graphObj && graphObj.mapElements) {
                                    graphObj.mapElements.each { el ->
                                        // Safely check for subflow element and non-JSONNull subflow object
                                        def isSubflowType = el.elementType && el.elementType.toString().equalsIgnoreCase('subflow')
                                        def hasSubflowObj = el.subflow != null && !(el.subflow instanceof net.sf.json.JSONNull)

                                        if (isSubflowType && hasSubflowObj) {
                                            def subflowObj = el.subflow
                                            def subflowId = (subflowObj.id != null && !(subflowObj.id instanceof net.sf.json.JSONNull)) ? (subflowObj.id as String) : null
                                            if (subflowId && !visitedFlows.contains(subflowId)) {
                                                visitedFlows.add(subflowId)
                                                discoveredSubflows.add(subflowId)
                                                queue.add(subflowId)
                                                def subflowName = (subflowObj.developerName != null && !(subflowObj.developerName instanceof net.sf.json.JSONNull)) ? subflowObj.developerName : subflowId
                                                echo "🔍 Discovered Subflow: '${subflowName}' (${subflowId}) under parent flow ${currentFlowId}"
                                            }
                                        }
                                    }
                                }
                            } else {
                                echo "⚠️ Note: Could not fetch flow graph for ${currentFlowId} (Status: ${graphResponse.status}). Proceeding without subflow expansion."
                            }
                        }

                        def allFlowIds = [] as List
                        allFlowIds.addAll(discoveredSubflows)
                        initialFlowIds.each { id ->
                            if (!allFlowIds.contains(id)) {
                                allFlowIds.add(id)
                            }
                        }

                        echo "Target Identity Provider ID: ${targetIdpId}"
                        echo "Updating IDP for ${allFlowIds.size()} flow(s) (${discoveredSubflows.size()} subflow(s), ${initialFlowIds.size()} root flow(s))..."

                        // Step 2: Update IDP on each flow targeting the latest snapshot version to preserve all canvas elements/subflows
                        allFlowIds.each { flowId ->
                            echo "=== Fetching latest snapshot definition for Flow ID: ${flowId} ==="

                            // Step 2a: Lookup latest snapshot to ensure we target the newest version (with subflows)
                            def snapResponse = httpRequest(
                                httpMode: 'GET',
                                ignoreSslErrors: true,
                                url: "${flowBaseUrl}/api/draw/1/flow/snap/${flowId}",
                                customHeaders: commonHeaders,
                                validResponseCodes: '100:599',
                                consoleLogResponseBody: false
                            )

                            def latestVersionId = null
                            def latestEditingToken = null

                            if (snapResponse.status < 300 && snapResponse.content) {
                                def snapshots = readJSON(text: snapResponse.content)
                                if (snapshots instanceof List && !snapshots.isEmpty()) {
                                    def latestSnapshot = snapshots.max { it.dateCreated }
                                    if (latestSnapshot && latestSnapshot.id && latestSnapshot.id.versionId && !(latestSnapshot.id.versionId instanceof net.sf.json.JSONNull)) {
                                        latestVersionId = latestSnapshot.id.versionId as String
                                    }
                                    if (latestSnapshot && latestSnapshot.editingToken && !(latestSnapshot.editingToken instanceof net.sf.json.JSONNull)) {
                                        latestEditingToken = latestSnapshot.editingToken as String
                                    }
                                }
                            }

                            // Step 2b: Fetch flow definition
                            def getFlowResponse = httpRequest(
                                httpMode: 'GET',
                                ignoreSslErrors: true,
                                url: "${flowBaseUrl}/api/draw/1/flow/${flowId}",
                                customHeaders: commonHeaders,
                                validResponseCodes: '100:599',
                                consoleLogResponseBody: true
                            )

                            if (getFlowResponse.status >= 300 || !getFlowResponse.content) {
                                error("Failed to fetch Flow definition for Flow ID: ${flowId}. Status: ${getFlowResponse.status}")
                            }

                            def flowObj = readJSON(text: getFlowResponse.content)

                            // Explicitly bind flowObj to the latest snapshot version and editing token to prevent reverting canvas elements
                            if (latestVersionId) {
                                if (!flowObj.id || (flowObj.id instanceof net.sf.json.JSONNull)) {
                                    flowObj.id = [:]
                                }
                                flowObj.id.versionId = latestVersionId
                                echo "Binding flow update to latest snapshot version: ${latestVersionId}"
                            }
                            if (latestEditingToken) {
                                flowObj.editingToken = latestEditingToken
                            }

                            if (!flowObj.identityProvider || (flowObj.identityProvider instanceof net.sf.json.JSONNull)) {
                                flowObj.identityProvider = [:]
                            }
                            flowObj.identityProvider.id = targetIdpId
                            if (flowObj.identityProvider.allowedGroups == null || (flowObj.identityProvider.allowedGroups instanceof net.sf.json.JSONNull)) {
                                flowObj.identityProvider.allowedGroups = []
                            }
                            if (flowObj.identityProvider.allowedUsers == null || (flowObj.identityProvider.allowedUsers instanceof net.sf.json.JSONNull)) {
                                flowObj.identityProvider.allowedUsers = []
                            }

                            def flowName = (flowObj.developerName != null && !(flowObj.developerName instanceof net.sf.json.JSONNull)) ? flowObj.developerName : flowId
                            echo "Updating Flow: ${flowName} (${flowId}) with Identity Provider ID: ${targetIdpId}"

                            writeJSON file: 'flow_update_payload.json', json: flowObj
                            def payloadJson = readFile file: 'flow_update_payload.json'

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

                            echo "✅ Successfully updated Identity Provider for Flow: ${flowName} (${flowId})"
                        }
                        echo "✅ All Flow IDP updates completed successfully."
                    }
                }
            }
        }

        stage('Publish Master & Subflows') {
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

                        // Step 1: Discover any subflows connected to the master flow
                        def queue = [masterFlowId] as List
                        def visitedFlows = [masterFlowId] as Set
                        def subflowIds = [] as List

                        while (!queue.isEmpty()) {
                            def currentFlowId = queue.remove(0)
                            def graphResponse = httpRequest(
                                httpMode: 'GET',
                                ignoreSslErrors: true,
                                url: "${flowBaseUrl}/api/draw/2/graph/flow/${currentFlowId}",
                                customHeaders: commonHeaders,
                                validResponseCodes: '100:599',
                                consoleLogResponseBody: false
                            )

                            if (graphResponse.status < 300 && graphResponse.content) {
                                def graphObj = readJSON(text: graphResponse.content)
                                if (graphObj && graphObj.mapElements) {
                                    graphObj.mapElements.each { el ->
                                        def isSubflowType = el.elementType && el.elementType.toString().equalsIgnoreCase('subflow')
                                        def hasSubflowObj = el.subflow != null && !(el.subflow instanceof net.sf.json.JSONNull)

                                        if (isSubflowType && hasSubflowObj) {
                                            def subflowObj = el.subflow
                                            def subflowId = (subflowObj.id != null && !(subflowObj.id instanceof net.sf.json.JSONNull)) ? (subflowObj.id as String) : null
                                            if (subflowId && !visitedFlows.contains(subflowId)) {
                                                visitedFlows.add(subflowId)
                                                subflowIds.add(subflowId)
                                                queue.add(subflowId)
                                                def subflowName = (subflowObj.developerName != null && !(subflowObj.developerName instanceof net.sf.json.JSONNull)) ? subflowObj.developerName : subflowId
                                                echo "🔍 Discovered Subflow to publish: '${subflowName}' (${subflowId})"
                                            }
                                        }
                                    }
                                }
                            }
                        }

                        // Step 2: Publish all subflows first
                        if (subflowIds && !subflowIds.isEmpty()) {
                            echo "Publishing ${subflowIds.size()} subflow(s) before master flow..."
                            subflowIds.each { subflowId ->
                                echo "=== Publishing Subflow ID: ${subflowId} ==="
                                def snapResponse = httpRequest(
                                    httpMode: 'GET',
                                    ignoreSslErrors: true,
                                    url: "${flowBaseUrl}/api/draw/1/flow/snap/${subflowId}",
                                    customHeaders: commonHeaders,
                                    validResponseCodes: '100:599',
                                    consoleLogResponseBody: true
                                )

                                if (snapResponse.status >= 300 || !snapResponse.content) {
                                    error("Failed to list snapshots for Subflow ID: ${subflowId}. Status: ${snapResponse.status}")
                                }

                                def snapshots = readJSON(text: snapResponse.content)
                                if (!(snapshots instanceof List) || snapshots.isEmpty()) {
                                    error("No snapshots found for Subflow ID: ${subflowId}")
                                }

                                def latestSnapshot = snapshots.max { it.dateCreated }
                                def versionId = (latestSnapshot?.id?.versionId != null && !(latestSnapshot.id.versionId instanceof net.sf.json.JSONNull)) ? (latestSnapshot.id.versionId as String) : null

                                if (!versionId) {
                                    error("Could not determine version ID for Subflow ID: ${subflowId}")
                                }

                                echo "Latest version for Subflow ${subflowId}: ${versionId} (created ${latestSnapshot.dateCreated})"

                                def activateResponse = httpRequest(
                                    httpMode: 'POST',
                                    ignoreSslErrors: true,
                                    url: "${flowBaseUrl}/api/draw/1/flow/activation/${subflowId}/${versionId}/true/true",
                                    customHeaders: commonHeaders,
                                    validResponseCodes: '100:599',
                                    consoleLogResponseBody: true
                                )

                                if (activateResponse.status >= 300) {
                                    error("Publish failed for Subflow ID: ${subflowId}. Status=${activateResponse.status}")
                                }
                                echo "✅ Subflow ID ${subflowId} published successfully (version ${versionId})"
                            }
                        }

                        // Step 3: Publish master flow
                        echo "=== Publishing Master Flow ID: ${masterFlowId} ==="
                        def snapResponse = httpRequest(
                            httpMode: 'GET',
                            ignoreSslErrors: true,
                            url: "${flowBaseUrl}/api/draw/1/flow/snap/${masterFlowId}",
                            customHeaders: commonHeaders,
                            validResponseCodes: '100:599',
                            consoleLogResponseBody: true
                        )

                        if (snapResponse.status >= 300 || !snapResponse.content) {
                            error("Failed to list snapshots for Master Flow ID: ${masterFlowId}. Status: ${snapResponse.status}")
                        }

                        def snapshots = readJSON(text: snapResponse.content)
                        if (!(snapshots instanceof List) || snapshots.isEmpty()) {
                            error("No snapshots found for Master Flow ID: ${masterFlowId}")
                        }

                        def latestSnapshot = snapshots.max { it.dateCreated }
                        def versionId = (latestSnapshot?.id?.versionId != null && !(latestSnapshot.id.versionId instanceof net.sf.json.JSONNull)) ? (latestSnapshot.id.versionId as String) : null

                        if (!versionId) {
                            error("Could not determine version ID for Master Flow ID: ${masterFlowId}")
                        }

                        echo "Latest version for Master Flow ID ${masterFlowId}: ${versionId} (created ${latestSnapshot.dateCreated})"

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
 