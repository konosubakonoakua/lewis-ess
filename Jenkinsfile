@Library('ecdc-pipeline')
import ecdcpipeline.ContainerBuildNode
import ecdcpipeline.PipelineBuilder

project = "lewis"

container_build_nodes = [
  'centos7': new ContainerBuildNode('dockerregistry.esss.dk/ecdc_group/build-node-images/centos7-build-node:10.0.0-dev', '/usr/bin/scl enable devtoolset-11 rh-python38 -- /bin/bash -e -x')
]


// Define number of old builds to keep.
num_artifacts_to_keep = '1'

// Set number of old builds to keep.
properties([[
  $class: 'BuildDiscarderProperty',
  strategy: [
    $class: 'LogRotator',
    artifactDaysToKeepStr: '',
    artifactNumToKeepStr: num_artifacts_to_keep,
    daysToKeepStr: '',
    numToKeepStr: num_artifacts_to_keep
  ]
]]);

// Set periodic trigger at 3:56 every day.
properties([
  pipelineTriggers([cron('56 3 * * *')]),
])

pipeline_builder = new PipelineBuilder(this, container_build_nodes)
pipeline_builder.activateEmailFailureNotifications()

builders = pipeline_builder.createBuilders { container ->
  pipeline_builder.stage("${container.key}: Checkout") {
    dir(pipeline_builder.project) {
      scm_vars = checkout scm
    }
    container.copyTo(pipeline_builder.project, pipeline_builder.project)
  }  // stage

  pipeline_builder.stage("${container.key}: 3.8") {
    container.sh """
      pyenv local 3.8
      which python
      python --version
      python -m pip install --user -r ${project}/requirements-dev.txt
    """
  } // stage

   pipeline_builder.stage("${container.key}: Test") {
     def test_output = "TestResults.xml"
     container.sh """
       pyenv local 3.7 3.8 3.9 
       which python
       python --version
       cd ${project}
       python -m tox -- --junitxml=${test_output}
     """
     container.copyFrom("${project}/${test_output}", ".")
     xunit thresholds: [failed(unstableThreshold: '0')], tools: [JUnit(deleteOutputFiles: true, pattern: '*.xml', skipNoTestFiles: false, stopProcessingIfError: true)]
   } // stage
}  // createBuilders

node {
  dir("${project}") {
    scm_vars = checkout scm
  }

  try {
    parallel builders
  } catch (e) {
    throw e
  }

  // Delete workspace when build is done
  cleanWs()
}

