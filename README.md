## TDR Generated GraphQL

This project generates, packages and deploys to the sonatype nexus the case classes generated by the [sbt-graphql](https://github.com/muuki88/sbt-graphql) project.

These classes will be used by other TDR repositories to communicate with the consignment API.

### Building locally
* Add a new query file to `src/main/graphql`
* Add a schema file generated by the [consignment api](https://github.com/nationalarchives/tdr-consignment-api) project. This will be generated in the `target/sbt-graphql` directory and should be copied into `src/main/resources/schema.graphql`
* Bump the version in `build.sbt`
* Run `sbt package publishLocal`

### Releasing
* Commit and push the new query file in `src/main/graphql` Don't commit `src/main/resources/schema.graphql`
* Make sure the version is bumped in `build.sbt`
* Merge to master. This project will be built by the deploy job for the consignment API but it can be also run manually. Go to the TDR Graphql Code Generation
job in Jenkins, paste the schema from the consignemnt api repo into the build parameters and build.