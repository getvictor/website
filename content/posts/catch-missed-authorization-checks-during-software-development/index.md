+++
title = 'Catch missed authorization checks during software development'
description = "Authorization checks in Fleet's Go codebase"
date = 2023-11-10
categories = ["Software Development", "Security"]
tags = ["Authorization", "Golang", "Application Security"]
draft = false
+++

{{< youtube jbkPLQpzPtc >}}

Authorization is giving permission to a user to do an action on the server.
As developers, we must ensure that users are only allowed to do what they are authorized.

One way to ensure that authorization has happened is to loudly flag when it hasn't.
This is how we do it at [Fleet Device Management](https://www.fleetdm.com).

In our code base, we use the [go-kit library](https://github.com/go-kit/kit). Most of the general endpoints are created
in the [handler.go](https://github.com/fleetdm/fleet/blob/36421bd5055d37a4c39a04e0f9bd96ad47951131/server/service/handler.go#L729) file. For example:

```go
// user-authenticated endpoints
ue := newUserAuthenticatedEndpointer(svc, opts, r, apiVersions...)

ue.POST("/api/_version_/fleet/trigger", triggerEndpoint, triggerRequest{})
```

Every endpoint calls **kithttp.NewServer** and wraps the endpoint with our **AuthzCheck**.
From [handler.go](https://github.com/fleetdm/fleet/blob/36421bd5055d37a4c39a04e0f9bd96ad47951131/server/service/handler.go#L729):

```go
e = authzcheck.NewMiddleware().AuthzCheck()(e)
return kithttp.NewServer(e, decodeFn, encodeResponse, opts...)
```

{{< figure src="AuthzCheck.jpg" alt="Catch missed authorization check block diagram" >}}

This means that after the business logic is processed, the AuthzCheck is called. 
This check ensures that authorization was checked. Otherwise, an error is returned. 
From [authzcheck.go](https://github.com/fleetdm/fleet/blob/36421bd5055d37a4c39a04e0f9bd96ad47951131/server/service/middleware/authzcheck/authzcheck.go#L51):

```go
// If authorization was not checked, return a response that will
// marshal to a generic error and log that the check was missed.
if !authzctx.Checked() {
    // Getting to here means there is an authorization-related bug in our code.
    return nil, authz.CheckMissingWithResponse(response)
}
```

This additional check is useful during our development and QA process, to ensure that authorization always happens in our business logic.

_This article originally appeared in [Fleet's blog](https://fleetdm.com/guides/catch-missed-authorization-checks-during-software-development)._
