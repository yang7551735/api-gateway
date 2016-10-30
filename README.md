|网站名称                      |网址                               |
|-----------------------------|-----------------------------------|
| 3scale	| https://www.3scale.net/api-management/ |
| akana	| https://www.akana.com/solutions/api-gateway |
|ApiAxle|	http://apiaxle.com/|
|Apigee |	https://apigee.com/|
|Beluga（中文）|	https://github.com/restran/api-gateway |
|IBM azure API网关（中文）	https://azure.microsoft.com/zh-cn/services/api-management/ https://azure.microsoft.com/zh-cn/services/application-gateway/ |
|Kong |	https://getkong.org/ |
|Mashery API Management	https://www.mashery.com/api-management |
|mulesoft	| https://www.mulesoft.com/platform/api/manager |
|Orange （中文）|	http://orange.sumory.com/ |
| rackla |	http://rackla.org |
|strongloop |	https://strongloop.com/node-js/api-gateway/ |
|Tenso |	http://avoidwork.github.io/tenso/ |
|Tyk |	https://tyk.io/ |
|WSO2 API Manager | 	http://wso2.com/products/api-manager/ |
| 亚马逊API网关（中文）	| https://aws.amazon.com/cn/api-gateway/?nc1=h_ls |
| 其他 |	https://github.com/fagongzi/gateway |
| 阿里云API 网关（中文） |	https://www.aliyun.com/product/apigateway |
 	 

    public void testGateway(GatewayRequest gatewayRequest) {

        AccessToken accessToken = null;

        String sessionId = null;

        String[] scopes = null;

        ThirdPartPolicy thirdPartAppPolicy = thirdPartAppService.getPolicyByAppKey(gatewayRequest.getAppKey());

        ApiPolicy apiPolicy = apiService.getApiPolicy(gatewayRequest.getMethod(), gatewayRequest.getVersion());



        //request-transformer,请求格式转换



        if (thirdPartAppPolicy.allowIp(gatewayRequest.getRemoteIp())) {
            //异常
        }

        //有的IP只允许特定IP访问
        if(apiPolicy.allowIp(gatewayRequest.getRemoteIp())){

        }

        //校验授权，内网请求不用授权
        if (apiPolicy.isNeedAccessToken()) {
            accessToken = tokenService.obtainToken(gatewayRequest.getAccessToken());
            sessionId = accessToken.getSessionId();
            scopes = accessToken.getScopes();
        }

        //防篡改
        if (globalSetting.isNeedSignCheck() || !gatewayRequest.isFromIntranet()) {
            signChecker.check(gatewayRequest.getRequestParams());
        }

        //判断用户是否授权给第三方访问这个API
        if(scopeChecker.checkAllowAccess(scopes,gatewayRequest.getMethod())){

        }

        //判断第三方应用是否可以使用这个API
        if (!thirdPartAppPolicy.canAccessApi(gatewayRequest.getMethod())) {

        }

        //限流（sessionId不校验，app+method）
        rateLimitChecker.checkAndRecord(apiPolicy,thirdPartAppPolicy);

        //全局sessionId的限流

        //全局请求Request限流

        //异常日志通知

        //response-transformer

    }
