### Image registry
Image registries should be part of any container plan -- if you can use dockerhub great, but if you need something internal you should use it.

Currently Docker supports four solutions. Internal registries, docker hub, docker store, and docker cloud.

Swarm can connect directly with extra configuration!

### Docker Hub Advanced :O
Most popular image registry - but it does have some extra (lightweight) image building.

Linking GitHub/BitBucket to DockerHub and chaining images together will be in this doc :)

dockerhub has many of the same ACL's you might have for gh repos. THere are webhooks etc. just like a GitHub repo.

Organizations are a thing, you can access people and add/remove as you do in GH

DockerHub does poll GitHub and BitBucket if you so please. It'll build images on the fly as needed from changes on specfic branches within a git repository.

You can also set up build settings when you depend on other images through FROM. Build triggers might be needed, and you can open up POST URL's on dockerhub to build automagically when those endpoints are whacked.

### Docker registry v2
Docker/Distrobution is the de facto private docker registry.

It's open source, supports auth and certificates. There's not any way to handle acl's etc or a web UI. Nice thing is being able to support backend storage with s3/azure/gcp/alibaba etc.

`docker run -f -p 5000:5000 -v registry-vol:/var/lib/registry -name registry registry` <- it's just the `registry image`. you can tag things with the prefix `my-registry:5000/name-of-image`

hosting a registry on swarm, all docker nodes would advertise on 127.0.0.1:500 :O


