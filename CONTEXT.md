# Tanned

Tanned connects a full frontend framework to a backend without making the backend responsible for the web interface.

## Language

**Frontend**:
The TanStack-based web system that owns browser routes, navigation, redirects, rendering, and requests for backend data.
_Avoid_: Frontend application, client application

**Backend**:
The server-side system that exposes an API containing the product's data and server-enforced rules, including authentication and authorization.
_Avoid_: Backend application, server application

**Tanned**:
The integration between the frontend and backend that removes repeated authentication and API-client scaffolding while preserving frontend ownership of the web experience.
_Avoid_: Frontend adapter, backend adapter
