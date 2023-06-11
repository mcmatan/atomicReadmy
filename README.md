// Retry configuration used for workflow steps
const retryConfig = { count: 10, backOff: 1000 };

// Base part of the workflow URLs
const workflowUrlBase = "/workflows/userCreation/steps";

// Your application API endpoint which creates a user
async function createUserEndpoint(idempotencyKey: guid, userName: string, email: string) {
    const payload = {userName, email};
    // Triggers workflow and handles retries and timeouts
    const userCreatedResponse = await postWorkflow("createUser", "/workflows/userCreation/index", idempotencyKey, payload, {startWorkFlow: true, timeout: 8000});

    return {
        status: 200,
        userId: userCreatedResponse.id
    }
}

/*

        The following URLs will be invoked via Atomic!

 */

// User creation workflow, triggered at POST /workflows/userCreation/index
async function createUserWorkflow({idempotencyKey: guid, payload: any}) {
    const userCreatedResponse = await performWorkflowStep("createUser", idempotencyKey, payload);

    await performWorkflowStep("sendEmail", idempotencyKey, userCreatedResponse.response);

    await performWorkflowStep("sendAnalytics", idempotencyKey, userCreatedResponse.response);
}

// Creates a user in the database, triggered at POST /workflows/userCreation/steps/createUser
async function createUserDBStep({idempotencyKey: guid, payload: any}) {
    await db.saveUserToDB(idempotencyKey, payload);
}

// Sends a user creation email, triggered at POST /workflows/userCreation/steps/sendEmail
async function createUserEmailStep({idempotencyKey: guid, payload: any}) {
    await emailClient.sendEmail(idempotencyKey, payload);
}

// Sends analytics data for user creation, triggered at POST /workflows/userCreation/steps/sendAnalytics
async function createUserAnalyticsStep({idempotencyKey: guid, payload: any}) {
    await analytics.userCreated(idempotencyKey, payload);
}

/*

    Helper functions:

 */


// Helper function to POST requests to the atomic workflow server
async function postWorkflow(workflowName: string, stepInvokeUrl: string, idempotencyKey: guid, payload: any, options: any = {}) {
    return await httpClient.post(`${atomicBaseUrl}/${clientId}`, {
        workflowName,
        stepInvokeUrl: `${myBaseUrl}${stepInvokeUrl}`,
        idempotencyKey,
        payload,
        ...options,
    });
}

// Helper function to perform a workflow step
async function performWorkflowStep(stepName: string, idempotencyKey: guid, payload: any) {
    return await postWorkflow("createUser", `${workflowUrlBase}/${stepName}`, idempotencyKey, payload, {retry: retryConfig});
}
