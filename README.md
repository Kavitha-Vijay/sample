using System;
using System.Net;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Text;
using System.Threading.Tasks;
using System.Xml.Serialization;
using System.IO;

class Program
{
    static async Task Main(string[] args)
    {
        // Proxy configuration
        var proxy = new WebProxy
        {
            Address = new Uri("http://your-proxy-server:your-proxy-port"),
            BypassProxyOnLocal = false,
            UseDefaultCredentials = false,
            Credentials = new NetworkCredential("your-username", "your-password")
        };

        // HttpClientHandler with proxy and cookies
        var handler = new HttpClientHandler
        {
            Proxy = proxy,
            UseProxy = true,
            UseCookies = true,
            CookieContainer = new CookieContainer()
        };

        using (var httpClient = new HttpClient(handler))
        {
            // Set custom header ALM: false
            httpClient.DefaultRequestHeaders.Add("ALM", "false");

            // Authentication endpoint and payload
            var authUrl = "https://your_octane_server_url/authentication/sign_in";
            var authPayload = new
            {
                client_id = "your_client_id",
                client_secret = "your_client_secret"
            };

            var authContent = new StringContent(
                Newtonsoft.Json.JsonConvert.SerializeObject(authPayload),
                Encoding.UTF8,
                "application/json"
            );

            try
            {
                // Send authentication request
                var authResponse = await httpClient.PostAsync(authUrl, authContent);
                authResponse.EnsureSuccessStatusCode();

                // Extract and print the cookies
                var cookies = handler.CookieContainer.GetCookies(new Uri(authUrl));
                foreach (Cookie cookie in cookies)
                {
                    Console.WriteLine($"Auth Cookie: {cookie.Name} = {cookie.Value}");
                }

                // Test results endpoint and payload
                var testResultsUrl = "https://your_octane_server_url/api/shared_spaces/{shared_space_id}/workspaces/{workspace_id}/test-results";
                var testResultsPayload = new TestResultsPayload
                {
                    Name = "Automated Test Run",
                    Status = "Passed",
                    Test = new Test
                    {
                        Type = "test_automated",
                        Id = "1001"
                    },
                    Started = "2024-05-21T08:00:00Z",
                    Duration = 120,
                    TestRunNativeStatus = "Passed",
                    CiJob = new CiJob
                    {
                        Type = "ci_job",
                        Id = "2001"
                    }
                };

                var testResultsContent = new StringContent(
                    SerializeToXml(testResultsPayload),
                    Encoding.UTF8,
                    "application/xml"
                );

                // Ensure the content type is set correctly
                testResultsContent.Headers.ContentType = new MediaTypeHeaderValue("application/xml");

                // Send test results request
                var testResultsResponse = await httpClient.PostAsync(testResultsUrl, testResultsContent);
                testResultsResponse.EnsureSuccessStatusCode();

                // Read and print the response
                var testResultsResponseBody = await testResultsResponse.Content.ReadAsStringAsync();
                Console.WriteLine($"Test Results Response Body: {testResultsResponseBody}");
            }
            catch (HttpRequestException e)
            {
                Console.WriteLine($"Request error: {e.Message}");
            }
        }
    }

    // Serialize an object to XML string
    private static string SerializeToXml<T>(T value)
    {
        if (value == null)
        {
            throw new ArgumentNullException(nameof(value));
        }

        var xmlSerializer = new XmlSerializer(typeof(T));
        using (var stringWriter = new StringWriter())
        {
            xmlSerializer.Serialize(stringWriter, value);
            return stringWriter.ToString();
        }
    }
}

// Classes for XML serialization
[XmlRoot("TestResults")]
public class TestResultsPayload
{
    [XmlElement("Name")]
    public string Name { get; set; }

    [XmlElement("Status")]
    public string Status { get; set; }

    [XmlElement("Test")]
    public Test Test { get; set; }

    [XmlElement("Started")]
    public string Started { get; set; }

    [XmlElement("Duration")]
    public int Duration { get; set; }

    [XmlElement("TestRunNativeStatus")]
    public string TestRunNativeStatus { get; set; }

    [XmlElement("CiJob")]
    public CiJob CiJob { get; set; }
}

public class Test
{
    [XmlElement("Type")]
    public string Type { get; set; }

    [XmlElement("Id")]
    public string Id { get; set; }
}

public class CiJob
{
    [XmlElement("Type")]
    public string Type { get; set; }

    [XmlElement("Id")]
    public string Id { get; set; }
}

