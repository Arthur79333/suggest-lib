# suggest-lib
suggest-lib Multibranch pipeline with correct version based on branch

**Jenkinsfile stages (pseudocode)**

    (when release/*) calculate correct version based on branch & last tag  
    compile  
    test  
    (when release/* or main) publish  
    (when release/*) git tag  
