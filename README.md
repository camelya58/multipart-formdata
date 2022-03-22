# multipart-formdata
Description for Multipart/form-data POST request for Swagger 2.0

If you want to upload file with multiple form-data having different content-type you need to use Content-Type: multipart/form-data.
```
curl -X POST 'http:\\host:post\api'\
-H 'Content-Type: multipart/form-data; charset=utf-8'\
-F 'file=@sample.pdf;type=application/pdf'\
-F 'meta={
          "author": "Akunin"
    };type=application/json;charset=utf-8'
```

```
POST /form.html HTTP/1.1
Host: server.com
Referer: http://server.com/form.html
User-Agent: Mozilla
Content-Type: multipart/form-data; boundary=-------------573cf973d5228
Content-Length: 288
Connection: keep-alive
Keep-Alive: 300
(пустая строка)
(отсутствующая преамбула)
---------------573cf973d5228
Content-Disposition: form-data; name="meta"
Content-Type: application/json

text
---------------573cf973d5228
Content-Disposition: form-data; name="file"; filename="sample.pdf"
Content-Type: application/pdf
```
You need to create a controller with @RequestPart annotations for parameters.
```java
@RestController
public class UploadController {

  private final String storageUrl = "C:\storage"; //some path

  @PostMapping(value="/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE, produces = MediaType.APPLICATION_JSON_VALUE)
  ResponseEntity<String> uploadDocument(
                              @ApiParam(value = "{\"author\":\"string\"}", required = true) // only for Swagger 2.0
                              @RequestPart("meta") DocumentInfoDto documentInfoDto,
                              @RequestPart("file") MultipartFile file) {
    try {
      // do something with documentInfoDto 
      Files.copy(file.getInputStream(), Paths.get(storageUrl).resolve(file.getOriginalName);
      return ResponseEntity.status(HttpStatus.OK)
            .body(
    } catch (Exception e) {
      return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body("Failed to upload the file");
    }
  }
}
```

For Swagger 3.0 this is enought. https://swagger.io/docs/specification/describing-request-body/file-upload/

The whole request will have a common responseBody and you can can your "meta" as an object.

But for Swagger 2.0 the "meta" will be as a parameter which type is "String". https://swagger.io/docs/specification/2-0/file-upload/

So, if you want to receive your object you need to create custom JsonDeserializer (remember to handle deserialization errors) and custom AbstractJackson2HttpMessageConverter
(to avoid the exception - "Content-Type: multipart/form-data; boundary=---WebKitFormBoundary7MA4YWxkTrZu0gW utf-8 not supported")
```java
public class DocumentInfoDtoDeserializer extends StdDeserializer<DocumentInfoDto> {

  public DocumentInfoDtoDeserializer() {
    this(null);
  }

  public DocumentInfoDtoDeserializer(Class<?> vc) {
    super(null);
  }
  
  @Override
  public DocumentInfoDto deserialize(JsonParser jp, DeserializationContext context) throws IOException {
    JsonNode node = jp.getCodec().readTree(jp);
    String author = node.get("author").toString();
    return new DocumentInfoDto(author);
  }
```

Add the annotation @JsonDeserialize(using = DocumentInfoDtoDeserializer.class) to a class.
```java
@Data
@AllArgsConstructor
@JsonDeserialize(using = DocumentInfoDtoDeserializer.class)
public class DocumentInfoDto {

  String author;
}
```
```java
@Component
public class MultipartJackson2HttpMessageConverter extends AbstractJackson2HttpMessageConverter {

  /**
  * Converter for support http request with header Content-Type: multipart/form-data
  */
  public MultipartJackson2HttpMessageConverter(ObjectMapper objectMapper) {
    super(objectMapper, MediaType.APPLICATION_OCTET_STREAM);
  }

  @Override
  public boolean canWrite(Class<?> clazz, MediaType mediaType) {
    return false;
  }

  @Override
  public boolean canWrite(Type type, Class<?> clazz, MediaType mediaType) {
    return false;
  }

  @Override
  protected boolean canWrite(MediaType mediaType) {
    return false;
  }
}
```
