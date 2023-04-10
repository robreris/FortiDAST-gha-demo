# DemoFrontEnd

This is the sample repo we will use to hold all Hugo Settings/Config/templates for Fortinet CSE TECRecipies.

To start, clone this repo.

The Hugo page built by GitActions (every time this repo is updated) is here: https://fortinetsecdevops.github.io/DemoFrontEnd/, and it explains in a few steps how to get started and how to interact with Hugo to build a statib website with your workshop/TECREcipie guide

To build and run via docker locally:
```sh
docker build -t demo-frontend .
docker run -p 1313:1313 demo-frontend:latest
```

In a browser, navigate to: http://localhost:1313/DemoFrontEnd/
