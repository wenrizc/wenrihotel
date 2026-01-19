• 我按代码里实际出现的签名/加解密逻辑归纳了每个项目“逆向”的做法和功能（只基于当前代码可见部分）：
                                                                                                           
  - order-takeaway/src/elemeGetSign.js：手写 MD5 实现，签名输入是 token&timestamp&appKey&data（见末尾注    
    释），输出 32 位小写 MD5。order-takeaway/index.food.js 和 order-takeaway/spiders/spiderEleme.js 用它生 
    成 Ele.me 的 sign 参数，拼到请求 URL 上，用于通过服务端验签。                                          
  - order-takeaway/socket.js：WebSocket 传输前后做 AES 加/解密，密钥是 UTC 日期 yyyyMMdd，实现的是与服务端 
    通讯的“协议加密”，防止明文传输。                                                                       
  - order/socket.js：同 order-takeaway，用日期作为 AES 密钥对 WebSocket 消息加/解密，保证客户端与服务端交互    一致。                                                                                                 
  - order/index.browser.js：打包了 CryptoJS（AES/MD5/HMAC 等），本身是库实现，供上面的 socket 加解密使用， 
    不是单独“逆向逻辑”。                                                                                   
  - shopee_group2/lib/md5.js：先 md5 参数串，再拼接固定盐 55b03 生成二次 md5，返回 55b03-<md5> 形式，属于自    定义签名。                                                                                             
  - shopee_group2/lib/generateShopeeTasks.js、shopee_group2/lib/parser/category.js、shopee_group2/lib/     
    parser/list.js：构造 Shopee API 参数串 encodeStr 并计算 encode，把签名随任务元数据传递，供后续请求使   
    用，属于“复刻客户端签名”。                                                                             
  - shopee_group2/src/shopeeSign.js：通过 jsdom 模拟浏览器环境，加载高度混淆的签名逻辑（含 MD5/AES），用于 
    生成 Shopee 的签名/令牌，属于“浏览器环境复刻 + 混淆代码执行”的逆向方式。                               
  - sp/src/app/in_sign.js：完整复刻 MD5 算法，签名流程为：MD5(curtime) + data + MD5(固定 key) 拼接后再做   
    MD5，输出 2.x<md5><curtime>。这是典型“客户端签名算法复刻”。                                            
  - sp/src/food/i.meituan.encrypt.js：仿造美团 rohrdata 反爬参数。代码伪造 window/document，收集浏览器尺寸/    分辨率/URL/输入轨迹等，生成 rohr 对象后用 pako.deflate + base64 加密；同时对请求参数排序后同样压缩编码 
    生成 sign。功能是模拟前端生成的反爬字段。
  - sp/src/auto/nio/nio.decrypt.js：AES-128-ECB 解密 base64 数据（有 Java 代码注释对应 AES/ECB/            
    PKCS5Padding），用于还原 NIO 加密响应。                                                                
  - sp/src/auto/nio/aes.decrypt.js：AES-CBC 加/解密，依据 key 长度选择 128/256，用于通用 NIO 数据加解密。  
  - sp/src/paidknowledge/zhihu.sign.js：将参数（排除 sign）按 key 排序后拼接，末尾加盐，做 MD5，生成签名， 
    用于知乎接口验签。                                                                                     
  - sp/src/ecommerce/dewuh5.decryption.js：同上风格，排序拼接 + MD5 + 可选盐，生成 Dewu H5 签名。          
  - crawler/src/seenreq/index.ts、crawler/src/seenreq/seenreq.ts：MD5 用于请求去重的“签名/哈希键”，不是反爬    逆向。                                                                                                 
  - order-utils/export/shipment.new.tmall.js：仅匹配到 SIGN 文本，没有实际加解密/签名实现。                
                                                                                                     