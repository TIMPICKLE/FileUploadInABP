# FileUpload
 
## 纯后端就是这样，但是如果你使用了ABP  CLI Generate proxy的话，会有一些问题

### 问题是这样的，我后端想接受一个文件，很自然的写的类型是 IFormFile file。然后又很自然的使用ABP CLI生成我需要的前端代理。

### 然后，就，代理报错，显示无法导出类型 IFormFile。

Incorrect proxy generated for file upload API · Issue #6384 · abpframework/abp · GitHub
https://github.com/abpframework/abp/issues/6384


 同样的，swagger接口测试时候显示可以上传没毛病，甚至可以多个上传文件。但是在Angular里显示的无法使用IFormFile类型。（忽然觉得还是JS比较自由自在）

评论中的解决方案是不使用Proxy中生成的服务，而是自己写http请求。这样做法相当不好。首先你跑到线上环境还是要重新跑proxy的，生产环境爆错直接挂。第二，服务器地址或者端口会变，这样到时候错了还要手动去改端口，很麻烦。

直接抛我的解决方案：

I also meet this issue，so there is an alternative solutions. when you upload a file, don't use "IFromFile" type. try to create another Dto, and one of its properties is " public string fileContent{ get;set; }". like this

public class ReportsUploadDto
    {
        public string Name
        {
            get;
            set;    
        }

        public string FileType
        {
            get;
            set;
        }

        public string LinkedCourseId
        {
            get;
            set;
        }

        public string FileContent
        {
            get;
            set;
        }
    }

so in frontend(i use angular). generate a proxy base on ABP, than follow the "ReportsUploadDto" type. After that, the string obtained from Base64 should be converted into byte and stored in minio.

我的comment意思是，不要传文件格式。另外建一个Dto（文件传输对象）。文件内容以Base64 string放进去，放到FileContent属性内。这样完整的传到后端，再做Convert解码为流文件。（因为minio接受流文件），如果你使用其他分布式文件系统，按要求来就行了，二进制流是万能的。

        public async Task<List<ThirdPartReportsDto>> UploadThirdPartReportsListWithByteForm(ThirdPartReportsUploadDto[] files)
        {
            // List上传
            var output = new List<ThirdPartReportsDto>();
            foreach (var file in files)
            {
                var id = Guid.NewGuid();
                var newFile = new ThirdPartReportsEntity(id, 0, file.FileType, CurrentTenant.Id, file.Name, file.LinkedCourseId);
                var created = await _repository.InsertAsync(newFile);
                var base64Str = file.FileContent.Split("base64,")[1];
                byte[] byteResult = Convert.FromBase64String(base64Str);
                await _blobContainer.SaveAsync(id.ToString(), byteResult, true).ConfigureAwait(false);
                output.Add(ObjectMapper.Map<ThirdPartReportsEntity, ThirdPartReportsDto>(newFile));
            }
            return output;
        }

 注意，一开始转码报错，我看了看base64的文件头要掐掉，不然符合格式。


另外一些细节：存文件信息到数据库中，一定得按Guid做任何增删改查操作，因为你永远不知道用户会不会做出脑溢血操作，比如同文件名。

更多细节:
​https://blog.csdn.net/dongnihao/article/details/125057181?spm=1001.2014.3001.5502
