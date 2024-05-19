# rpc调用最佳实践
### 原始代码
```java
public class UserRpcClient {
    public static String SERVICE_NAME="user-service" ;

    public User getById(Long userId) throws RpcException {
        JSONObject response = null;
        try {
            Map<String, String> params = new HashMap<String, String>(2);
            params.put("id", userId);
            response = doGet(SERVICE_NAME, "getById", params);
        } catch (Exception e) {
            throw new RpcException("call getById failed", ErrorCode.RPC_UNEXPECTED_ERROR, e);
        }

        if (RpcConstants.RESP_STATUS_OK.equals(response.getString(RpcConstants.RESP_STATUS_KEY))) {
            JSONObject content = response.getJSONObject(RpcConstants.RESP_CONTENT_KEY);
            return content.toJavaObject(User.class);
        } else {
            log.warn("[getById] failed  ==========>  call user service resp: {}", response.toJSONString());
            throw new RpcException("status failed", ErrorCode.RPC_RESULT_FAILED_ERROR);
        }
    }
}

```
### 问题
处理网络异常、解析json、判断status状态、处理errorcode都是样板代码，在每一个client方法中都需要重复

### 优化
```java

public class RpcProxy {
    @Autowired
    private RawRpc rawRpc;

    //1.提取公共逻辑
    public <T> T call(Request request, TypeReference<T> typeReference, RpcHandler<T> rpcHandler) {
        RpcResponse response;
        try {
            response = rawRpc.call(request, RpcResponse.class);
            if (response.success()) {
                return response.parse(typeReference);
            }
            return rpcHandler.onFail(response.errorCode(), response.errorMsg());
        } catch (Exception e) {
            return rpcHandler.onException(e);
        }
    }
 
    @Data
    static class RpcResponse {
        public static final int STATUS_OK = 0;
 
        private int statusCode;
        private String content;
        private String errMsg;


 
        public boolean success() {
            return Objects.equals(STATUS_OK, statusCode);
        }
 
        public int errCode() {
            return statusCode;
        }

        public String errorMsg() {
            return  errorMsg;
        }
 
 
        public <T> T parse(TypeReference<T> typeReference) {
            if (typeReference.getType() == String.class) {
                return (T) content;
            }
            return JSON.parseObject(content, typeReference);
        }
    }

    // 封装变化
    public interface RpcHandler<T> {
        T onFail(int errCode, String errMsg);
        T onException(Exception ex);
    }
}

// demo 调用时只需关心反回类型，失败异常处理即可。
public class UserRpcClient {
    public static String SERVICE_NAME="user-service" ;
    private RpcProxy rpcProxy;

    public User getById(Long userId) throws RpcException {

        Request request = buildRequest(userId);
 
        User user = rpcProxy.call(request, new TypeReference<User>, new RpcProxy.RpcHandler<User>() {
 
            @Override
            public User onFail(int errCode, String errMsg) {
                log.error("call getById failed where userId={}, errCode={}, errMsg={}", userId, errCode, errMsg);
                return Collections.emptyList();
            }
 
            @Override
            public User onException(Exception ex) {
                log.error("all getById exp  where userId={}",e)
                return Collections.emptyList();;
            }
        });
    }
}

```
