package mailhog


/*
dependencies:
    testImplementation("org.testcontainers:testcontainers:1.15.2")
    testImplementation(platform("org.http4k:http4k-bom:4.3.5.4"))
    testImplementation("org.http4k:http4k-core")
    testImplementation("org.http4k:http4k-client-apache") {
        because("mailhog http api messages access")
    }
    testImplementation("org.http4k:http4k-format-jackson") {
        because("mailhog messages access unmarshalling")
    }
    testImplementation("com.sun.mail:javax.mail:1.5.6") {
        because("mailhog messages decoding")
    }
 */
import com.fasterxml.jackson.annotation.JsonInclude
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.module.kotlin.KotlinModule
import mailhog.MailhogJackson.auto
import org.http4k.client.ApacheClient
import org.http4k.core.Body
import org.http4k.core.Method
import org.http4k.core.Request
import org.http4k.format.ConfigurableJackson
import org.testcontainers.containers.GenericContainer
import org.testcontainers.containers.wait.strategy.Wait
import javax.mail.internet.MimeUtility

/**
 * A test container for mailhog, to allow to test sending emails with a real smtp server.
 *
 * Example of use:
 * <code>
 *     @Container
 *     val mailhog = MailhogContainer()
 *
 *     @Test fun `my_test`() {
 *       myMailServiceUnderTest.setupSmtp(mailhog.smtpHost, mailhog.smtpPort)
 *
 *       myMailServiceUnderTest.sendAMail()
 *
 *       val messages = mailhog.messages
 *       expectThat(messages).hasSize(1)
 *
 *       expectThat(messages)
 *           .filter { it.to.map { it.address }.contains("test1@email.com") }
 *           .hasSize(1).first().and {
 *               get { subject }.contains("My Test")
 *               get { body }.contains("Hello!")
 *           }
 *     }
 * </code>
 */
class MailhogContainer: GenericContainer<MailhogContainer>("mailhog/mailhog") {
    private val PORT_SMTP = 1025
    private val PORT_HTTP = 8025

    init {
        withExposedPorts(PORT_SMTP, PORT_HTTP)
        waitingFor(Wait.forHttp("/"))
    }

    val smtpPort get() = getMappedPort(PORT_SMTP)
    val smtpHost get() = getContainerIpAddress()
    val httpUrl get() = "http://${getContainerIpAddress()}:${getMappedPort(PORT_HTTP)}"

    val messages get() = messagesBodyLens(client(Request(Method.GET, httpUrl + "/api/v2/messages")))
        .items?.map { it.toMailMessage() }?:listOf()

    companion object {
        val messagesBodyLens = Body.auto<MailhogMessages>().toLens()
        val client = ApacheClient()
    }
}

// message structure for easy common use case testing
data class MailMessage(val from:EmailAddress, val to:List<EmailAddress>, val subject:String, val body:String)
data class EmailAddress(val address:String)

fun MailhogMessage.toMailMessage() = MailMessage(
    from = From?.toEmailAddress()?:throw IllegalArgumentException("no from address"),
    to = (To?:listOf()).map { it.toEmailAddress() },
    subject = Content?.Headers?.get(MailhogMailHeaders.Subject.name)?.firstOrNull().orEmpty(),
    body = Content?.Body.orEmpty()
        .let { MimeUtility.decode(it.byteInputStream(), "quoted-printable").bufferedReader().readText() }
)

fun MailhogEmailAddress.toEmailAddress() = EmailAddress(Mailbox.orEmpty() + "@" + Domain.orEmpty())

// raw Mailhog data structure
data class MailhogMessages(val total: Number?, val count: Number?, val start: Number?, val items: List<MailhogMessage>?)

data class MailhogMessage(val ID: String?, val From: MailhogEmailAddress?, val To: List<MailhogEmailAddress>?, val Content: MailhogContent?, val Created: String?, val MIME: Any?, val Raw: MailhogRaw?)

data class MailhogEmailAddress(val Relays: Any?, val Mailbox: String?, val Domain: String?, val Params: String?)

data class MailhogContent(val Headers: Map<String, List<String>>, val Body: String?, val Size: Number?, val MIME: Any?)

data class MailhogRaw(val From: String?, val To: List<String>?, val Data: String?, val Helo: String?)

enum class MailhogMailHeaders {
    `Content-Transfer-Encoding`, `Content-Type`, Date, From,
    `MIME-Version`, `Message-ID`, Received, `Return-Path`, Subject, To
}

object MailhogJackson : ConfigurableJackson(
    ObjectMapper()
    .registerModule(KotlinModule())
    .deactivateDefaultTyping()
    .setSerializationInclusion(JsonInclude.Include.NON_NULL)
)
