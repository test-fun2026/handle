# handle
catch (Exception ex) {

    throw TokenizationExceptionTranslator.translate(ex);
}
----

import com.fasterxml.jackson.core.JsonProcessingException;
import com.truist.core.common.exception.SdkException;

import lombok.extern.slf4j.Slf4j;

import org.apache.http.conn.ConnectTimeoutException;
import org.apache.http.conn.ConnectionPoolTimeoutException;

import software.amazon.awssdk.awscore.exception.AwsServiceException;
import software.amazon.awssdk.core.exception.ApiCallAttemptTimeoutException;
import software.amazon.awssdk.core.exception.ApiCallTimeoutException;
import software.amazon.awssdk.core.exception.RetryableException;
import software.amazon.awssdk.http.HttpExecuteException;
import software.amazon.awssdk.services.sts.model.StsException;

import javax.net.ssl.SSLException;
import javax.net.ssl.SSLHandshakeException;

import java.io.InterruptedIOException;
import java.net.MalformedURLException;
import java.net.NoRouteToHostException;
import java.net.SocketException;
import java.net.SocketTimeoutException;
import java.net.URISyntaxException;
import java.net.UnknownHostException;
import java.security.cert.CertificateException;

@Slf4j
public final class TokenizationExceptionTranslator {

    private TokenizationExceptionTranslator() {
    }

    public static SdkException translate(Exception ex) {

        log.error("Tokenization failure occurred", ex);

        // AWS SERVICE EXCEPTIONS
        // Handled separately because status code inspection is needed

        if (ex instanceof AwsServiceException awsEx) {
            return handleAwsServiceException(awsEx);
        }

        return switch (ex) {

            // =========================================================
            // TIMEOUT EXCEPTIONS
            // =========================================================

            case ConnectTimeoutException e,
                 SocketTimeoutException e,
                 ApiCallTimeoutException e,
                 ApiCallAttemptTimeoutException e ->

                    new SdkException(
                            "408",
                            "Timeout while calling tokenization service",
                            "TOKENIZATION_TIMEOUT");

            // =========================================================
            // NETWORK / CONNECTIVITY FAILURES
            // =========================================================

            case UnknownHostException e,
                 NoRouteToHostException e,
                 HttpExecuteException e,
                 ConnectionPoolTimeoutException e,
                 RetryableException e,
                 SocketException e ->

                    new SdkException(
                            "503",
                            "Tokenization service unavailable",
                            "TOKENIZATION_UNAVAILABLE");

            // =========================================================
            // SECURITY / SSL FAILURES
            // =========================================================

            case SSLHandshakeException e,
                 SSLException e,
                 CertificateException e,
                 StsException e ->

                    new SdkException(
                            "401",
                            "Secure communication failure with tokenization service",
                            "TOKENIZATION_SECURITY_ERROR");

            // =========================================================
            // VALIDATION / PAYLOAD FAILURES
            // =========================================================

            case JsonProcessingException e,
                 IllegalArgumentException e ->

                    new SdkException(
                            "400",
                            "Invalid tokenization request",
                            "TOKENIZATION_BAD_REQUEST");

            // =========================================================
            // CONFIGURATION FAILURES
            // =========================================================

            case URISyntaxException e,
                 MalformedURLException e ->

                    new SdkException(
                            "500",
                            "Tokenization configuration failure",
                            "TOKENIZATION_CONFIGURATION_ERROR");

            // =========================================================
            // INTERRUPTED THREADS
            // =========================================================

            case InterruptedException e -> {

                Thread.currentThread().interrupt();

                yield new SdkException(
                        "503",
                        "Tokenization request interrupted",
                        "TOKENIZATION_INTERRUPTED");
            }

            case InterruptedIOException e -> {

                Thread.currentThread().interrupt();

                yield new SdkException(
                        "503",
                        "Tokenization I/O interrupted",
                        "TOKENIZATION_INTERRUPTED");
            }

            // =========================================================
            // FALLBACK
            // =========================================================

            default -> new SdkException(
                    "500",
                    "Unexpected tokenization failure",
                    "TOKENIZATION_INTERNAL_ERROR");
        };
    }

    private static SdkException handleAwsServiceException(
            AwsServiceException awsEx) {

        int statusCode = awsEx.statusCode();

        log.error(
                "AWS service exception occurred. statusCode={}, errorCode={}, message={}",
                statusCode,
                awsEx.awsErrorDetails() != null
                        ? awsEx.awsErrorDetails().errorCode()
                        : "UNKNOWN",
                awsEx.getMessage(),
                awsEx);

        // =========================================================
        // THROTTLING
        // =========================================================

        if (statusCode == 429) {

            return new SdkException(
                    "429",
                    "Tokenization service throttling",
                    "TOKENIZATION_THROTTLED");
        }

        // =========================================================
        // SERVER ERRORS
        // =========================================================

        if (statusCode >= 500) {

            return new SdkException(
                    String.valueOf(statusCode),
                    "Tokenization downstream service failure",
                    "TOKENIZATION_SERVER_ERROR");
        }

        // =========================================================
        // AUTHORIZATION FAILURES
        // =========================================================

        if (statusCode == 401 || statusCode == 403) {

            return new SdkException(
                    String.valueOf(statusCode),
                    "Unauthorized tokenization service request",
                    "TOKENIZATION_AUTHORIZATION_ERROR");
        }

        // =========================================================
        // BAD REQUEST FAILURES
        // =========================================================

        if (statusCode >= 400) {

            return new SdkException(
                    String.valueOf(statusCode),
                    "Invalid request sent to tokenization service",
                    "TOKENIZATION_BAD_REQUEST");
        }

        // =========================================================
        // FALLBACK
        // =========================================================

        return new SdkException(
                "500",
                "Unexpected AWS tokenization service failure",
                "TOKENIZATION_AWS_SERVICE_ERROR");
    }
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
