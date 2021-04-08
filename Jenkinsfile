library(
    identifier: 'pipeline-lib@4.8.0',
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
        def rootWAFACLARN = terraformOutputs.eks_cluster_waf_acl_arn.value

        sh("""#!/bin/bash
            set -ex
            helm init --client-only
            helm repo add scdp https://datastillery.github.io/charts
            helm repo update
            helm upgrade --install discovery-streams scdp/discovery-streams  \
                --version 0.3.1 \
                --namespace=discovery \
                --set ingress.enabled="true" \
                --set ingress.scheme="${ingressScheme}" \
                --set ingress.subnets="${subnets}" \
                --set ingress.securityGroups="${allowInboundTrafficSG}" \
                --set ingress.dnsZone="${dnsZone}" \
                --set ingress.root_dns_zone="${rootDnsZone}" \
                --set ingress.certificateARN="${certificateARNs}" \
                --set ingress.waf_acl_arn="${rootWAFACLARN}" \
                --values=discovery-streams.yaml \
                ${extraArgs}
        """.trim())
    }
}
