# handle
catch (Exception ex) {
  throw translateTokenizationException(ex);
}
-----
private SdkException translateTokenizationException(Exception ex) {

    log.error("Tokenization failure", ex);

    // TIMEOUTS

    if (ex instanceof ConnectTimeoutException ||
        ex instanceof SocketTimeoutException ||
        ex instanceof ApiCallTimeoutException ||
        ex instanceof ApiCallAttemptTimeoutException) {

        return new SdkException(
                "408",
                "Timeout while calling tokenization service",
                "TOKENIZATION_TIMEOUT");
    }

    // THROTTLING

    if (ex instanceof AwsServiceException awsEx &&
            awsEx.statusCode() == 429) {

        return new SdkException(
                "429",
                "Tokenization service throttling",
                "TOKENIZATION_THROTTLED");
    }

    // SERVER ERRORS

    if (ex instanceof AwsServiceException awsEx &&
            awsEx.statusCode() >= 500) {

        return new SdkException(
                String.valueOf(awsEx.statusCode()),
                "Tokenization downstream service failure",
                "TOKENIZATION_SERVER_ERROR");
    }

    // NETWORK FAILURES

    if (ex instanceof UnknownHostException ||
        ex instanceof NoRouteToHostException ||
        ex instanceof HttpExecuteException ||
        ex instanceof ConnectionPoolTimeoutException ||
        ex instanceof RetryableException ||
        ex instanceof SocketException) {

        return new SdkException(
                "503",
                "Tokenization service unavailable",
                "TOKENIZATION_UNAVAILABLE");
    }

    // SECURITY FAILURES

    if (ex instanceof SSLHandshakeException ||
        ex instanceof SSLException ||
        ex instanceof CertificateException ||
        ex instanceof StsException) {

        return new SdkException(
                "401",
                "Secure communication failure with tokenization service",
                "TOKENIZATION_SECURITY_ERROR");
    }

    // VALIDATION / PAYLOAD FAILURES

    if (ex instanceof JsonProcessingException ||
        ex instanceof IllegalArgumentException) {

        return new SdkException(
                "400",
                "Invalid tokenization request",
                "TOKENIZATION_BAD_REQUEST");
    }

    // CONFIGURATION ISSUES

    if (ex instanceof URISyntaxException ||
        ex instanceof MalformedURLException) {

        return new SdkException(
                "500",
                "Tokenization configuration failure",
                "TOKENIZATION_CONFIGURATION_ERROR");
    }

    // INTERRUPTED THREADS

    if (ex instanceof InterruptedException ||
        ex instanceof InterruptedIOException) {

        Thread.currentThread().interrupt();

        return new SdkException(
                "503",
                "Tokenization request interrupted",
                "TOKENIZATION_INTERRUPTED");
    }

    // FALLBACK

    return new SdkException(
            "500",
            "Unexpected tokenization failure",
            "TOKENIZATION_INTERNAL_ERROR");
}
-----
try {
    int status = Integer.parseInt(ex.getErrorCode());

    return status == 408 ||
           status == 429 ||
           status >= 500;

} catch (NumberFormatException e) {

    return false;
}
