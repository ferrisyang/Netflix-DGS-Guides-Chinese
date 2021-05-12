# 文件上传

在 GraphQL 中，你可以模拟一个文件上传操作，作为一个从客户端到 DGS 的 GraphQL mutation request。

下面的场景，描述了你如何使用一个 Multipart POST request 实现文件的上传和下载。对于文件上传和最佳实践的更多信息，请看来自 Apollo 博客 Khalil Stemmler 撰写的 [Apollo Server File Upload Best Practices](https://www.apollographql.com/blog/apollo-server-file-upload-best-practices-1e7f24cdc050)

## Multipart File Upload

一个 multipart request 就是一个带有 multiple parts 的 HTTP 请求：mutation 请求，文件数据，JSON 对象和其他。你可以使用 Apollo 的 Upload 客户端，或者甚至一个简单的 cURL，使用一个你在你的 schema 中，作为一个 Mutation 模拟的 multipart request 来发送一个文件数据的流。

![file-upload-multipart](../.gitbook/assets/file-upload-multipart.png)

使用 GraphQL mutations 用于上传文件的 multipart `POST` 请求更多的参数，请参照 [GraphQL multipart request specification](https://github.com/jaydenseric/graphql-multipart-request-spec) 。

DGS 框架支持 `Upload` scalar，在你的 mutation query 中作为一个 `MultipartFile` 来指定。当你发送文件上传的 multipart request 时，框架将处理每个部分并组装最终的GraphQL查询，并将其交给 data fetcher 进行进一步处理。

下面是一个将文件上传到 DGS 的 Mutation 查询示例：

```scheme
scalar Upload

extend type Mutation  {
    uploadScriptWithMultipartPOST(input: Upload!): Boolean
}
```

注意，你需要在你的 Schema 中声明 `Upload` scalar，尽管实现是由 DGS 框架提供的。在你的 DGS 里，添加一个 datafetcher 去处理这个 `MultipartFile`，如下所示：

```java
@DgsData(parentType = DgsConstants.MUTATION.TYPE_NAME, field = "uploadScriptWithMultipartPOST")
    public boolean uploadScript(DataFetchingEnvironment dfe) throws IOException {
        // NOTE: Cannot use @InputArgument  or Object Mapper to convert to class, because MultipartFile cannot be
        // deserialized
        MultipartFile file = dfe.getArgument("input");
        String content = new String(file.getBytes());
        return ! content.isEmpty();
    }
```

注意你不能使用 Jackson Object Mapper 去反序列化包含了 `MultipartFile` 的类型，你需要在你的输入中显式的获得文件参数。

在你的客户端，你可以使用 `apollo-upload-client` 去发送你的作为一个 multipart `POST` 包含文件数据的 Mutation query。下面是怎样配置你的 link：

```kotlin
import { createUploadLink } from 'apollo-upload-client'

const uploadLink = createUploadLink({ uri: uri })

const authedClient = authLink && new ApolloClient({
        link: uploadLink)),
        cache: new InMemoryCache()
})
```

一旦你设置好，创建你的 Mutation query 并且作为一个变量发送用户选择的文件：

```kotlin
// query for file uploads using multipart post
const UploadScriptMultipartMutation_gql = gql`
  mutation uploadScriptWithMultipartPOST($input: Upload!) {
    uploadScriptWithMultipartPOST(input: $input)
  }
`;

function MultipartScriptUpload() {
  const [
    uploadScriptMultipartMutation,
    {
      loading: mutationLoading,
      error: mutationError,
      data: mutationData,
    },
  ] = useMutation(UploadScriptMultipartMutation_gql);
  const [scriptMultipartInput, setScriptMultipartInput] = useState<any>();

  const onSubmitScriptMultipart = () => {
    const fileInput = scriptMultipartInput.files[0];
    uploadScriptMultipartMutation({
      variables: { input: fileInput },
    });
  };

  return (
    <div>
      <h3> Upload script using multipart HTTP POST</h3>
      <form
        onSubmit={e => {
          e.preventDefault();
          onSubmitScriptMultipart();
        }}>
        <label>
          <input
            type="file"
            ref={ref => {
              setScriptMultipartInput(ref!);
            }}
          />
        </label>
        <br />
        <br />
        <button type="submit">Submit</button>
      </form>
    </div>
  );
}
```

