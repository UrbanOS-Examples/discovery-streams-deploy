library(
    identifier: 'pipeline-lib@4.6.1',
    retriever: modernSCM([$class: 'GitSCMSource',
                          remote: 'https://github.com/SmartColumbusOS/pipeline-lib',
                          credentialsId: 'jenkins-github-user'])
)

properties([
    pipelineTriggers([scos.dailyBuildTrigger()]),
    parameters([
        booleanParam(defaultValue: false, description: 'Deploy to development environment?', name: 'DEV_DEPLOYMENT'),
        string(defaultValue: 'development', description: 'Image tag to deploy to dev environment', name: 'DEV_IMAGE_TAG')
    ])
])

def doStageIf = scos.&doStageIf
def doStageIfDeployingToDev = doStageIf.curry(env.DEV_DEPLOYMENT == "true")
def doStageIfMergedToMaster = doStageIf.curry(scos.changeset.isMaster && env.DEV_DEPLOYMENT == "false")
def doStageIfRelease = doStageIf.curry(scos.changeset.isRelease)

node('infrastructure') {
    ansiColor('xterm') {
        scos.doCheckoutStage()
        if(env.BRANCH_NAME.matches("PR-\\d*")) {
            doDryRun()
        }
        doStageIfDeployingToDev('Deploy to Dev') {
            deployTo('dev', true, "--set image.tag=${env.DEV_IMAGE_TAG} --recreate-pods")
        }

        doStageIfMergedToMaster('Process Dev job') {
            scos.devDeployTrigger('discovery-streams')
        }

        doStageIfMergedToMaster('Deploy to Staging') {
            deployTo('staging', true)
            scos.applyAndPushGitHubTag('staging')
        }

        doStageIfRelease('Deploy to Production') {
            deployTo('prod', false)
            scos.applyAndPushGitHubTag('prod')
        }
    }
}

def deployTo(environment, internal, extraArgs = '') {
    if (environment == null) throw new IllegalArgumentException("environment must be specified")

    scos.withEksCredentials(environment) {
        def terraformOutputs = scos.terraformOutput(environment)
        def subnets = terraformOutputs.public_subnets.value.join(/\\,/)
        def allowInboundTrafficSG = terraformOutputs.allow_all_security_group.value
        def certificateARNs = [terraformOutputs.root_tls_certificate_arn.value,terraformOutputs.tls_certificate_arn.value].join(/\\,/)
        def ingressScheme = internal ? 'internal' : 'internet-facing'
        def dnsZone = terraformOutputs.internal_dns_zone_name.value
        def rootDnsZone = terraformOutputs.root_dns_zone_name.value

        sh("""#!/bin/bash
            set -e
            helm init --client-only
            helm repo add scdp https://smartcitiesdata.github.io/charts
            helm repo update
            helm upgrade --install scdp/discovery-streams  \
                --namespace=discovery \
                --set ingress.enabled="true" \
                --set ingress.scheme="${ingressScheme}" \
                --set ingress.subnets="${subnets}" \
                --set ingress.securityGroups="${allowInboundTrafficSG}" \
                --set ingress.dnsZone="${dnsZone}" \
                --set ingress.root_dns_zone="${rootDnsZone}" \
                --set ingress.certificateARN="${certificateARNs}" \
                 --values=discovery-streams.yaml \
                ${extraArgs}
        """.trim())
    }
}

def doDryRun(environment = "dev") {
    scos.withEksCredentials(environment) {
            sh("""#!/bin/bash
            set -e
            helm init --client-only
            helm repo add scdp https://smartcitiesdata.github.io/charts
            helm repo update
            helm template scdp/discovery-streams  \
                --namespace=discovery \
                --set ingress.enabled="true" \
                --set ingress.scheme="internal" \
                --set ingress.subnets="subnet-a" \
                --set ingress.securityGroups="sg-a" \
                --set ingress.dnsZone="smartos.example" \
                --set ingress.root_dns_zone="rootsmartos.example" \
                --set ingress.certificateARN="certa" \
                 --values=discovery-streams.yaml \
            > template.validate

            kubectl apply -f template.validate --dry-run=true
        """.trim())
    }
}