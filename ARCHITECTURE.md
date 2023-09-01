# Architecture

This document outlines the architecture of the AI Nutrition-Pro application, including system context, containers, and deployment views. The architecture is depicted using C4 diagrams for enhanced clarity.

## System Context diagram

```mermaid
C4Context
  title System Context diagram for AI Nutrition-Pro
  Enterprise_Boundary(b0, "AI Nutrition-Pro boundary") {
    System(anps1, "AI Nutrition-Pro API Service")
  }
  Enterprise_Boundary(b1, "OpenAI") {
    System_Ext(ChatGPT, "ChatGPT", "LLM")
  }
  Enterprise_Boundary(b2, "Meal Planner A") {
    Person_Ext(n1, "Dietitian A")
    System_Ext(apa, "Meal Planner A System")
  }
  Rel(anps1, ChatGPT, "Uses for LLM featured content creation", "REST")
  Rel(n1, apa, "Uses for diet creation")
  Rel(apa, anps1, "Uses for AI content generation", "REST")
```

- AI Nutrition-Pro is API backend application that uses OpenAI ChatGPT as LLM solution. It connects to OpenAI server using REST.
- Meal Planner A is one of many Meal Planner applications that can be integrated with AI Nutrition-Pro. It connects to AI Nutrition-Pro using REST.
- Dietitian A uses Meal Planner A for making diets. For example, dietitian can generate new diet introductions based on existing ones. LLMs can create this content following personal style of dietitian.

## Containers Context diagram

```mermaid
C4Container
    title Container diagram for AI Nutrition-Pro

    Container_Boundary(c0, "AI Nutrition-Pro") {
        Container(api_gateway, "API Gateway", "Kong", "Authentication of clients, filtering of input, rate limiting")
        Container(app_control_plane, "Web Control Plane", "Golang, AWS Elastic Container Service", "Provides control plane to onboard and manage clients, configuration and check billing data")
        ContainerDb(control_plan_db, "Control Plane Database", "Amazon RDS", "Stores all data related to control plan, tenants, billing")
        Container(backend_api, "API Application", "Golang, AWS Elastic Container Service", "Provides AI Nutrition-Pro functionality via API")
        ContainerDb(api_db, "API database", "Amazon RDS", "Stores dietitian' content samples, request and responses to LLM.")
        Person(admin, "Administrator", "Administrator of AI Nutrition-Pro application")
        Person(onboard_manager, "App Onboarding Manager", "Employee that is onboarding new Meal Planner applications to AI Nutrition-Pro application")
    }

    Person(ce1, "Meal Planner application manager", "Meal Planner employee designated to manage AI Nutrition-Pro integration")

    System_Ext(mealApp, "Meal Planner", "Application to create diets by dietitians")
    Person_Ext(n1, "Dietitian")

    System_Ext(chatgpt, "ChatGPT-3.5", "LLM")

    Rel(n1, mealApp, "Uses for diet creation")
    Rel(mealApp, api_gateway, "Uses for AI content generation", "HTTPS/REST")
    Rel(api_gateway, backend_api, "Uses for AI content generation", "HTTPS/REST")
    Rel(ce1, app_control_plane, "Manage billings, onboard new dietitians")
    Rel(admin, app_control_plane, "Configure system properties")
    Rel(onboard_manager, app_control_plane, "Creates new tenant. Onboarding new meal planner applications")
    Rel(backend_api, chatgpt, "Utilizes ChatGPT for LLM-featured content creation", "HTTPS/REST")

    Rel(app_control_plane, control_plan_db, "read/write data", "TLS")
    Rel(backend_api, api_db, "read/write data", "TLS")
```

### External systems and persons

| Name | Type | Description | Responsibilities |
| --- | --- | --- | --- |
| Meal Planner application | External system, web application | One of many Meal Planner applications that can be integrated with AI Nutrition-Pro. It connects to AI Nutrition-Pro using REST and HTTPS. | - uploads samples of dietitians' content to AI Nutrition-Pro <br/> - fetches AI generated results, e.g. diet introduction, from AI Nutrition-Pro | 
| Dietitian | External person | It's a customer of Meal Planner application. It's using Meal Planner to create diets for patients. It will see AI generated content directly in Meal Planner. | - creates diets <br> - consents to AI processing of data |
|  ChatGPT-3.5 | External system, API | It's OpenAI product, an LLM solution | It will be used to generate content based on provided samples. |
 
### AI Nutrition-Pro container context systems and persons

| Name | Type | Description | Responsibilities |
| --- | --- | --- | --- |
| Web Control Plane | Internal system, Web application | It's written using Golang and deployed as Docker container into AWS Elastic Container Service. It uses Control Plane Database to store data. It's used in 3 roles: Administrator, App Onboarding Manager, and Meal Planner application manager. | Provide control plane to onboard and manage clients, configuration and check billing data |
| Control Plane Database | Internal database, Amazon RDS instance | Database | storing data for Web Control Plane |
| API Gateway | Internal system, API Gateway | Kong API Gateway | - authentication <br> - rate limiting <br> - filtering of input |
| API Application | Internal system, API application | It's written using Golang and deployed as Docker container into AWS Elastic Container Service | Provides AI Nutrition-Pro functionality via API. |
| API database | Internal database, Amazon RDS instance | Stored data: samples of dietitians' content, requests, and responses to LLM. | Storing data for API Application |
| Administrator | Internal Person | Administrator of AI Nutrition-Pro application. | - manage server configuration <br> - resolve problems <br> |
| App Onboarding Manager | Internal Person | Employee that is onboarding new Meal Planner applications to AI Nutrition-Pro application. | - manage configuration of integrated Meal Planner application |
| Meal Planner application manager | Internal Person | Employee of Meal Planner | Manages AI Nutrition-Pro integration, e.g. configuration, api keys, billings. |

## Deployment diagram

For deployment, we will use Amazon AWS Cloud.

```mermaid
C4Deployment
  title AI Nutrition-Pro deployment to AWS Cloud diagram
  Deployment_Node(aws, "AWS account") {
    Deployment_Node(vpc, "VPC", "App VPC") {
      Deployment_Node(ecs, "AWS ECS", "App ECS") {
        Container(api_gateway, "API Gateway", "Kong")
        Container(api_app, "API Application", "Golang")
        Container(web_control_plane, "Web Control Plane", "Golang")
      }
    }
    ContainerDb(api_db, "API Database", "AWS RDS")
    ContainerDb(control_plane_db, "Control Plane Database", "AWS RDS")
  }

  Rel(api_gateway, api_app, "REST/HTTPS")
  Rel(api_app, api_db, "Native/TLS")
  Rel(web_control_plane, control_plane_db, "Native/TLS")
```

- API Gateway - deployed into AWS ECS service and into App VPC. It's Kong. Accessing API Application using REST/HTTPS protocol.
- API Application - deployed as Docker container into AWS ECS service and into App VPC. It's application created in Golang. Accessing API Database using Native/TLS protocol.
- Web Control Plane - deployed as Docker container into AWS ECS service and into App VPC. It's application created in Golang. Accessing Control Plane Database using Native/TLS protocol.
- API Database - instance of AWS RDS.
- Control Plane Database - instance of AWS RDS.
