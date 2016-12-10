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
 	 
   
API 调用统计

galileo

https://galileo.mashape.com/test-environment/logs/



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
    
    
    
    
    package cn.com.gome.cloud.openplatform.gateway.console.service;

import cn.com.gome.cloud.openplatform.gateway.console.domain.*;
import com.alibaba.dubbo.common.utils.StringUtils;
import org.elasticsearch.action.search.SearchRequestBuilder;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.action.search.SearchType;
import org.elasticsearch.client.transport.TransportClient;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.search.aggregations.Aggregation;
import org.elasticsearch.search.aggregations.AggregationBuilders;
import org.elasticsearch.search.aggregations.Aggregations;
import org.elasticsearch.search.aggregations.bucket.histogram.DateHistogramBuilder;
import org.elasticsearch.search.aggregations.bucket.histogram.InternalHistogram;
import org.elasticsearch.search.aggregations.bucket.terms.InternalTerms;
import org.elasticsearch.search.aggregations.bucket.terms.StringTerms;
import org.elasticsearch.search.aggregations.bucket.terms.Terms;
import org.elasticsearch.search.aggregations.metrics.avg.InternalAvg;
import org.joda.time.DateTime;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.*;

@Service("elasticSearchService")
public class ElasticSearchService {
    @Autowired
    TransportClient elasticClient;

    @Autowired
    HourlyApiInvokeCountService hourlyApiInvokeCountService;

    @Autowired
    IpInfoService ipInfoService;

    /**
     * 获取指定日期24小时api调用数据
     *
     * @return
     */
    public List<HourlyApiInvokeCount> getHourlyApiInvokeCountListByDate(Calendar time) {

        List<HourlyApiInvokeCount> list = new ArrayList<>();
        time.set(Calendar.HOUR_OF_DAY, 0);
        time.set(Calendar.MINUTE, 0);
        time.set(Calendar.SECOND, 0);
        time.set(Calendar.MILLISECOND, 0);
        DateFormat df = new SimpleDateFormat("yyyy-MM-dd");
        String index = df.format(time.getTime());
        boolean exists = elasticClient.admin().indices()
                .prepareExists("cloud_gw_log-" + index)
                .execute().actionGet().isExists();
        if (exists) {
            SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-" + index).setTypes("api_log")
                    .setSearchType(SearchType.QUERY_AND_FETCH).setSize(0);

            requestBuilder.setPostFilter(QueryBuilders.rangeQuery("@timestamp").to(time.getTime()));
            requestBuilder.addAggregation(AggregationBuilders.dateHistogram("count").field("@timestamp").interval(3600 * 1000).timeZone("Asia/Shanghai").minDocCount(0).extendedBounds(time.getTimeInMillis(), time.getTimeInMillis() + 3600 * 1000 * 24 - 1)
                    .subAggregation(AggregationBuilders.terms("api").field("apiName").subAggregation(AggregationBuilders.avg("time").field("time"))));


            SearchResponse response = requestBuilder.execute().actionGet();
            InternalHistogram<InternalHistogram.Bucket> hourlyCounts = response.getAggregations().get("count");
            for (InternalHistogram.Bucket hourlyCount : hourlyCounts.getBuckets()) {
                Terms hourlyApiCounts = hourlyCount.getAggregations().get("api");
                DateTime date = (DateTime) hourlyCount.getKey();
                long millis = date.getMillis();
                for (Terms.Bucket hourlyApiCount : hourlyApiCounts.getBuckets()) {
                    long apiCount = hourlyApiCount.getDocCount();
                    String api = hourlyApiCount.getKey().toString();
                    HourlyApiInvokeCount hourlyApiInvokeCount = new HourlyApiInvokeCount();
                    hourlyApiInvokeCount.setApi(api);
                    hourlyApiInvokeCount.setCount(apiCount);
                    Date invokeDate = new Date(millis);
                    hourlyApiInvokeCount.setDate(invokeDate);
                    InternalAvg avgTime = hourlyApiCount.getAggregations().get("time");
                    hourlyApiInvokeCount.setErrorCount(getApiHourlyErrorCountByDate(invokeDate, api));
                    hourlyApiInvokeCount.setAvgTime(avgTime.getValue());
                    list.add(hourlyApiInvokeCount);
                }
            }
        }
        return list;
    }


    /**
     * 获取指定日期24小时api调用数据
     *
     * @return
     */
    public List<HourlyAppInvokeCount> getHourlyAppInvokeCountListByDate(Calendar time) {
        List<HourlyAppInvokeCount> list = new ArrayList<>();
        time.set(Calendar.HOUR_OF_DAY, 0);
        time.set(Calendar.MINUTE, 0);
        time.set(Calendar.SECOND, 0);
        time.set(Calendar.MILLISECOND, 0);
        DateFormat df = new SimpleDateFormat("yyyy-MM-dd");
        String index = df.format(time.getTime());
        boolean exists = elasticClient.admin().indices()
                .prepareExists("cloud_gw_log-" + index)
                .execute().actionGet().isExists();
        if (exists) {
            SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-" + index).setTypes("api_log")
                    .setSearchType(SearchType.QUERY_AND_FETCH).setSize(0);

            requestBuilder.setPostFilter(QueryBuilders.rangeQuery("@timestamp").to(time.getTime()));
            requestBuilder.addAggregation(AggregationBuilders.dateHistogram("count").field("@timestamp").interval(3600 * 1000).timeZone("Asia/Shanghai").minDocCount(0).extendedBounds(time.getTimeInMillis(), time.getTimeInMillis() + 3600 * 1000 * 24 - 1)
                    .subAggregation(AggregationBuilders.terms("app").field("appkey").subAggregation(AggregationBuilders.avg("time").field("time"))));


            SearchResponse response = requestBuilder.execute().actionGet();
            InternalHistogram<InternalHistogram.Bucket> hourlyCounts = response.getAggregations().get("count");
            for (InternalHistogram.Bucket hourlyCount : hourlyCounts.getBuckets()) {
                Terms hourlyAppCounts = hourlyCount.getAggregations().get("app");
                DateTime date = (DateTime) hourlyCount.getKey();
                long millis = date.getMillis();
                for (Terms.Bucket hourlyAppCount : hourlyAppCounts.getBuckets()) {
                    long apiCount = hourlyAppCount.getDocCount();
                    String app = hourlyAppCount.getKey().toString();
                    HourlyAppInvokeCount hourlyAppInvokeCount = new HourlyAppInvokeCount();
                    hourlyAppInvokeCount.setApp(app);
                    hourlyAppInvokeCount.setCount(apiCount);
                    Date invokeDate = new Date(millis);
                    hourlyAppInvokeCount.setDate(invokeDate);
                    InternalAvg avgTime = hourlyAppCount.getAggregations().get("time");
                    hourlyAppInvokeCount.setErrorCount(getAppHourlyErrorCountByDate(invokeDate, app));
                    hourlyAppInvokeCount.setAvgTime(avgTime.getValue());
                    list.add(hourlyAppInvokeCount);
                }
            }
        }
        return list;
    }

    /**
     * 获取指定日期24小时api调用数据
     *
     * @param date
     * @return
     */
    public List<InvokeCount> getHourlyInvokeCountListByDate(Date date) {
        List<InvokeCount> list = new ArrayList<>();
        DateFormat df = new SimpleDateFormat("yyyy-MM-dd");
        String index = df.format(date.getTime());
        boolean exists = elasticClient.admin().indices()
                .prepareExists("cloud_gw_log-" + index)
                .execute().actionGet().isExists();
        if (exists) {
            Calendar calendar = Calendar.getInstance();
            calendar.setTime(date);
            calendar.set(Calendar.HOUR_OF_DAY, 0);
            calendar.set(Calendar.MINUTE, 0);
            calendar.set(Calendar.SECOND, 0);
            calendar.set(Calendar.MILLISECOND, 0);
            SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-" + index).setTypes("api_log")
                    .setSearchType(SearchType.QUERY_AND_FETCH).setSize(0);
            requestBuilder.addAggregation(AggregationBuilders.dateHistogram("count").field("@timestamp").interval(3600 * 1000).timeZone("Asia/Shanghai").minDocCount(0).extendedBounds(calendar.getTimeInMillis(), calendar.getTimeInMillis() + 3600 * 1000 * 24 - 1));
            SearchResponse response = requestBuilder.execute().actionGet();
            InternalHistogram<InternalHistogram.Bucket> hourlyCounts = response.getAggregations().get("count");
            for (InternalHistogram.Bucket hourlyCount : hourlyCounts.getBuckets()) {
                DateTime dateTime = (DateTime) hourlyCount.getKey();
                long millis = dateTime.getMillis();
                long count = hourlyCount.getDocCount();
                InvokeCount invokeCount = new InvokeCount();
                invokeCount.setCount(count);
                invokeCount.setTime(millis);
                list.add(invokeCount);
            }
        }
        return list;
    }

    /**
     * 获取指定日期各个城市调用量
     *
     * @param date
     * @return
     */
    public List<ClientCityInvokeCount> getClientCityCountListByDate(Date date) throws IOException {
        List<ClientCityInvokeCount> list = new ArrayList<>();
        DateFormat df = new SimpleDateFormat("yyyy-MM-dd");
        String index = df.format(date.getTime());
        boolean exists = elasticClient.admin().indices()
                .prepareExists("cloud_gw_log-" + index)
                .execute().actionGet().isExists();
        if (exists) {
            SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-" + index).setTypes("api_log")
                    .setSearchType(SearchType.QUERY_AND_FETCH).setSize(0);
            requestBuilder.addAggregation(AggregationBuilders.terms("count").field("client"));
            SearchResponse response = requestBuilder.execute().actionGet();
            StringTerms clientInvokeCounts = response.getAggregations().get("count");

            List<Terms.Bucket> clientInvokeCountsList = clientInvokeCounts.getBuckets();
            for (Terms.Bucket clientInvokeCount : clientInvokeCountsList) {
                Optional<String> city = ipInfoService.getIpInfo(clientInvokeCount.getKey().toString());
                if (city.isPresent()) {
                    ClientCityInvokeCount invokeCount = new ClientCityInvokeCount();
                    invokeCount.setCity(city.get());
                    invokeCount.setCount(clientInvokeCount.getDocCount());
                    list.add(invokeCount);
                }
            }

        }
        return list;
    }

    /**
     * 获取指定api,指定日期24小时调用量
     *
     * @param api  当api为空时查询所有api
     * @param date
     * @return
     */
    public List<InvokeCount> getHourlyInvokeCountListByDateAndApi(Date date, String api) {
        List<InvokeCount> list = new ArrayList<>();
        DateFormat df = new SimpleDateFormat("yyyy-MM-dd");
        String index = df.format(date);
        boolean exists = elasticClient.admin().indices()
                .prepareExists("cloud_gw_log-" + index)
                .execute().actionGet().isExists();
        if (exists) {
            SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-" + index).setTypes("api_log")
                    .setSearchType(SearchType.QUERY_AND_FETCH).setSize(0);
            if (StringUtils.isNotEmpty(api))
                requestBuilder.setQuery(QueryBuilders.termQuery("apiName", api));
            Calendar calendar = Calendar.getInstance();
            calendar.setTime(date);
            calendar.set(Calendar.HOUR_OF_DAY, 0);
            calendar.set(Calendar.MINUTE, 0);
            calendar.set(Calendar.SECOND, 0);
            calendar.set(Calendar.MILLISECOND, 0);

            requestBuilder.addAggregation(AggregationBuilders.dateHistogram("count").field("@timestamp").interval(3600 * 1000).minDocCount(0).timeZone("Asia/Shanghai").extendedBounds(calendar.getTimeInMillis(), calendar.getTimeInMillis() + 3600 * 1000 * 24 - 1));
            SearchResponse response = requestBuilder.execute().actionGet();
            InternalHistogram<InternalHistogram.Bucket> hourlyCounts = response.getAggregations().get("count");
            for (InternalHistogram.Bucket hourlyCount : hourlyCounts.getBuckets()) {
                DateTime dateTime = (DateTime) hourlyCount.getKey();
                long millis = dateTime.getMillis();
                long count = hourlyCount.getDocCount();
                InvokeCount invokeCount = new InvokeCount();
                invokeCount.setCount(count);
                invokeCount.setTime(millis);
                list.add(invokeCount);
            }
        }
        return list;
    }

    /**
     * 获取指定app,指定日期24小时调用量
     *
     * @param app  当api为空时查询所有app
     * @param date
     * @return
     */
    public List<InvokeCount> getHourlyInvokeCountListByDateAndApp(Date date, String app) {
        List<InvokeCount> list = new ArrayList<>();
        DateFormat df = new SimpleDateFormat("yyyy-MM-dd");
        String index = df.format(date);
        boolean exists = elasticClient.admin().indices()
                .prepareExists("cloud_gw_log-" + index)
                .execute().actionGet().isExists();
        if (exists) {
            SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-" + index).setTypes("api_log")
                    .setSearchType(SearchType.QUERY_AND_FETCH).setSize(0);
            if (StringUtils.isNotEmpty(app))
                requestBuilder.setQuery(QueryBuilders.termQuery("appkey", app));
            Calendar calendar = Calendar.getInstance();
            calendar.setTime(date);
            calendar.set(Calendar.HOUR_OF_DAY, 0);
            calendar.set(Calendar.MINUTE, 0);
            calendar.set(Calendar.SECOND, 0);
            calendar.set(Calendar.MILLISECOND, 0);

            requestBuilder.addAggregation(AggregationBuilders.dateHistogram("count").field("@timestamp").interval(3600 * 1000).minDocCount(0).timeZone("Asia/Shanghai").extendedBounds(calendar.getTimeInMillis(), calendar.getTimeInMillis() + 3600 * 1000 * 24 - 1));
            SearchResponse response = requestBuilder.execute().actionGet();
            InternalHistogram<InternalHistogram.Bucket> hourlyCounts = response.getAggregations().get("count");
            for (InternalHistogram.Bucket hourlyCount : hourlyCounts.getBuckets()) {
                DateTime dateTime = (DateTime) hourlyCount.getKey();
                long millis = dateTime.getMillis();
                long count = hourlyCount.getDocCount();
                InvokeCount invokeCount = new InvokeCount();
                invokeCount.setCount(count);
                invokeCount.setTime(millis);
                list.add(invokeCount);
            }
        }
        return list;
    }


    /**
     * 获取指定api的每日调用量
     *
     * @param api 当api为空时查询所有api
     * @return
     */
    public List<DaylyApiInvokeCount> getDaylyInvokeCountListByApi(String api) {

        List<DaylyApiInvokeCount> daylyApiInvokeCounts = new ArrayList<>();

        SearchRequestBuilder requestBuilder = elasticClient
                .prepareSearch("cloud_gw_log-*")
                .setTypes("api_log")
                .setSearchType(SearchType.QUERY_AND_FETCH)
                .setSize(0);

        if (StringUtils.isNotEmpty(api)){
            requestBuilder.setQuery(QueryBuilders.termQuery("apiName", api));
        }

        DateHistogramBuilder dateHistogramBuilder = AggregationBuilders
                .dateHistogram("count")
                .field("@timestamp")
                .interval(3600 * 1000 * 24)
                .timeZone("Asia/Shanghai")
                .minDocCount(1)
                .extendedBounds(0L, System.currentTimeMillis())
                .subAggregation(AggregationBuilders.avg("time").field("time"));

        requestBuilder.addAggregation(dateHistogramBuilder);

        SearchResponse response = requestBuilder.
                execute().
                actionGet();

        InternalHistogram<InternalHistogram.Bucket> DaylyCounts = response.getAggregations().get("count");

        for (InternalHistogram.Bucket daylyCount : DaylyCounts.getBuckets()) {
            DateTime dateTime = (DateTime) daylyCount.getKey();
            long count = daylyCount.getDocCount();
            if (count == 0)
                continue;
            DaylyApiInvokeCount invokeCount = new DaylyApiInvokeCount();
            invokeCount.setApi(StringUtils.isEmpty(api) ? null : api);
            invokeCount.setCount(count);
            invokeCount.setDate(dateTime.toDate());
            invokeCount.setErrorCount(getApiErrorCountByDate(dateTime.toDate(), api));
            InternalAvg avgTime = daylyCount.getAggregations().get("time");
            invokeCount.setAvgTime(avgTime.getValue());
            daylyApiInvokeCounts.add(invokeCount);
        }

        return daylyApiInvokeCounts;

    }
    /**
     * 获取指定app的每日调用量
     *
     * @param app 当app为空时查询所有app
     * @return
     */
    public List<DaylyAppInvokeCount> getDaylyInvokeCountListByApp(String app) {
        List<DaylyAppInvokeCount> list = new ArrayList<>();
        Calendar date = Calendar.getInstance();
        SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-*").setTypes("api_log")
                .setSearchType(SearchType.QUERY_AND_FETCH).setSize(0);
        if (StringUtils.isNotEmpty(app))
            requestBuilder.setQuery(QueryBuilders.termQuery("appkey", app));
        requestBuilder.addAggregation(AggregationBuilders.dateHistogram("count").field("@timestamp").interval(3600 * 1000 * 24).timeZone("Asia/Shanghai").minDocCount(1).extendedBounds(0L, date.getTimeInMillis())
                .subAggregation(AggregationBuilders.avg("time").field("time")));
        SearchResponse response = requestBuilder.execute().actionGet();
        InternalHistogram<InternalHistogram.Bucket> DaylyCounts = response.getAggregations().get("count");
        for (InternalHistogram.Bucket daylyCount : DaylyCounts.getBuckets()) {
            DateTime dateTime = (DateTime) daylyCount.getKey();
            long count = daylyCount.getDocCount();
            if (count == 0)
                continue;
            DaylyAppInvokeCount invokeCount = new DaylyAppInvokeCount();
            invokeCount.setApp(StringUtils.isEmpty(app) ? null : app);
            invokeCount.setCount(count);
            invokeCount.setDate(dateTime.toDate());
            invokeCount.setErrorCount(getAppErrorCountByDate(dateTime.toDate(), app));
            InternalAvg avgTime = daylyCount.getAggregations().get("time");
            invokeCount.setAvgTime(avgTime.getValue());
            list.add(invokeCount);
        }
        return list;
    }


    /**
     * 获取指定日期的当日调用量汇总数据
     *
     * @param date 指定日期
     * @return
     */
    public List<DaylyApiInvokeCount> getDaylyApiInvokeCountListByDate(Date date) {
        List<DaylyApiInvokeCount> list = new ArrayList<>();
        DateFormat df = new SimpleDateFormat("yyyy-MM-dd");
        SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-" + df.format(date)).setTypes("api_log")
                .setSearchType(SearchType.QUERY_AND_FETCH).setSize(0);
        requestBuilder.addAggregation(AggregationBuilders.terms("api").field("apiName").subAggregation(AggregationBuilders.avg("time").field("time")));
        SearchResponse response = requestBuilder.execute().actionGet();
        StringTerms apiCounts = response.getAggregations().get("api");
        for (Terms.Bucket a : apiCounts.getBuckets()) {
            String api = a.getKeyAsString();
            Long count = a.getDocCount();
            Double time = ((InternalAvg) a.getAggregations().get("time")).getValue();
            DaylyApiInvokeCount invokeCount = new DaylyApiInvokeCount();
            invokeCount.setApi(api);
            invokeCount.setErrorCount(getApiErrorCountByDate(date, api));
            invokeCount.setCount(count);
            invokeCount.setDate(date);
            invokeCount.setAvgTime(time);
            list.add(invokeCount);
        }
        return list;
    }

    /**
     * 获取指定api指定日期前30天的每日调用量
     *
     * @param api  当api为空时查询所有api
     * @param date
     * @return
     */
    public List<InvokeCount> getLast30DaylyInvokeCountListByApiAndDate(String api, Date date) {
        List<InvokeCount> list = new ArrayList<>();
        SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-*").setTypes("api_log")
                .setSearchType(SearchType.QUERY_AND_FETCH).setSize(0);
        if (StringUtils.isNotEmpty(api))
            requestBuilder.setQuery(QueryBuilders.termQuery("apiName", api));
        Calendar from = Calendar.getInstance();
        from.setTime(date);
        from.add(Calendar.DATE, -30);
        requestBuilder.addAggregation(AggregationBuilders.dateHistogram("count").field("@timestamp").interval(3600 * 1000 * 24).timeZone("Asia/Shanghai").extendedBounds(from.getTimeInMillis(), date.getTime()));
        SearchResponse response = requestBuilder.execute().actionGet();
        InternalHistogram<InternalHistogram.Bucket> hourlyCounts = response.getAggregations().get("count");
        for (InternalHistogram.Bucket hourlyCount : hourlyCounts.getBuckets()) {
            DateTime dateTime = (DateTime) hourlyCount.getKey();
            long millis = dateTime.getMillis();
            long count = hourlyCount.getDocCount();
            InvokeCount invokeCount = new InvokeCount();
            invokeCount.setCount(count);
            invokeCount.setTime(millis);
            list.add(invokeCount);
        }
        return list;
    }

    /**
     * 获取指定app指定日期前30天的每日调用量
     *
     * @param app  当app为空时查询所有app
     * @param date
     * @return
     */
    public List<InvokeCount> getLast30DaylyInvokeCountListByAppAndDate(String app, Date date) {
        List<InvokeCount> list = new ArrayList<>();
        SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-*").setTypes("api_log")
                .setSearchType(SearchType.QUERY_AND_FETCH).setSize(0);
        if (StringUtils.isNotEmpty(app))
            requestBuilder.setQuery(QueryBuilders.termQuery("appkey", app));
        Calendar from = Calendar.getInstance();
        from.setTime(date);
        from.add(Calendar.DATE, -30);
        requestBuilder.addAggregation(AggregationBuilders.dateHistogram("count").field("@timestamp").interval(3600 * 1000 * 24).timeZone("Asia/Shanghai").extendedBounds(from.getTimeInMillis(), date.getTime()));
        SearchResponse response = requestBuilder.execute().actionGet();
        InternalHistogram<InternalHistogram.Bucket> hourlyCounts = response.getAggregations().get("count");
        for (InternalHistogram.Bucket hourlyCount : hourlyCounts.getBuckets()) {
            DateTime dateTime = (DateTime) hourlyCount.getKey();
            long millis = dateTime.getMillis();
            long count = hourlyCount.getDocCount();
            InvokeCount invokeCount = new InvokeCount();
            invokeCount.setCount(count);
            invokeCount.setTime(millis);
            list.add(invokeCount);
        }
        return list;
    }


    /**
     * 获取指定api最近1小时内实时调用量
     *
     * @param apiName
     * @return
     */
    public List<InvokeCount> getRealTimeInvokeCountListByApi(String apiName) {
        List<InvokeCount> list = new ArrayList<>();
        Calendar time = Calendar.getInstance();
        SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-*").setTypes("api_log")
                .setSearchType(SearchType.QUERY_AND_FETCH).addFields("@timestamp", "apiName").setSize(0);
        requestBuilder.setQuery(QueryBuilders.rangeQuery("@timestamp").to(time.getTime()));
        requestBuilder.addAggregation(AggregationBuilders.dateHistogram("count").field("@timestamp").interval(5 * 1000).timeZone("Asia/Shanghai").minDocCount(0).extendedBounds(time.getTimeInMillis() - 3600 * 1000, time.getTimeInMillis()));
        SearchResponse response = requestBuilder.execute().actionGet();
        InternalHistogram<InternalHistogram.Bucket> hourlyCounts = response.getAggregations().get("count");
        for (InternalHistogram.Bucket hourlyCount : hourlyCounts.getBuckets()) {
            DateTime date = (DateTime) hourlyCount.getKey();
            long millis = date.getMillis();
            long count = hourlyCount.getDocCount();
            InvokeCount invokeCount = new InvokeCount();
            invokeCount.setCount(count);
            invokeCount.setTime(millis);
            list.add(invokeCount);
        }
        return list;
    }

    /**
     * 获取总调用数
     *
     * @return
     */
    public long getTotalCount() {
        SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-*").setTypes("api_log")
                .setSearchType(SearchType.QUERY_AND_FETCH).setSize(0);
        SearchResponse response = requestBuilder.execute().actionGet();
        return response.getHits().getTotalHits();
    }

    /**
     * 获取调用错误数
     *
     * @return
     */
    public long getErrorCount() {
        SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-*").setTypes("api_log")
                .setSearchType(SearchType.QUERY_AND_FETCH).setSize(0);
        requestBuilder.setQuery(QueryBuilders.existsQuery("errorCode"));
        SearchResponse response = requestBuilder.execute().actionGet();
        return response.getHits().getTotalHits();
    }

    /**
     * 获取总调用数
     *
     * @return
     */
    public long getTotalCountByDate(Date date) {
        DateFormat df = new SimpleDateFormat("yyyy-MM-dd");
        String index = df.format(date);
        boolean exists = elasticClient.admin().indices()
                .prepareExists("cloud_gw_log-" + index)
                .execute().actionGet().isExists();
        if (exists) {
            SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-" + df.format(date)).setTypes("api_log")
                    .setSearchType(SearchType.QUERY_AND_FETCH).setSize(0);
            SearchResponse response = requestBuilder.execute().actionGet();
            return response.getHits().getTotalHits();
        }
        return 0;
    }

    /**
     * 指定API在某一天的报错数
     *
     * @return
     */
    public long getApiErrorCountByDate(Date date, String api) {

        //获取时间戳
        DateFormat df = new SimpleDateFormat("yyyy-MM-dd");
        String index = df.format(date);

        boolean exists = elasticClient
                .admin()
                .indices()
                .prepareExists("cloud_gw_log-" + index)
                .execute()
                .actionGet()
                .isExists();

        if (exists) {
            SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-" + df.format(date)).setTypes("api_log")
                    .setSearchType(SearchType.QUERY_AND_FETCH).setSize(0);
            requestBuilder.setQuery(QueryBuilders.existsQuery("errorCode"));
            if (StringUtils.isNotEmpty(api)) {
                requestBuilder.setQuery(QueryBuilders.boolQuery().must(QueryBuilders.matchQuery("apiName", api)).must(QueryBuilders.existsQuery("errorCode")));
            }
            SearchResponse response = requestBuilder.execute().actionGet();
            return response.getHits().getTotalHits();
        }

        return 0;
    }

    /**
     * 获取app调用错误数
     *
     * @return
     */
    public long getAppErrorCountByDate(Date date, String app) {
        DateFormat df = new SimpleDateFormat("yyyy-MM-dd");
        String index = df.format(date);
        boolean exists = elasticClient.admin().indices()
                .prepareExists("cloud_gw_log-" + index)
                .execute().actionGet().isExists();
        if (exists) {
            SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-" + df.format(date)).setTypes("api_log")
                    .setSearchType(SearchType.QUERY_AND_FETCH).setSize(0);
            requestBuilder.setQuery(QueryBuilders.existsQuery("errorCode"));
            if (StringUtils.isNotEmpty(app)) {
                requestBuilder.setQuery(QueryBuilders.boolQuery().must(QueryBuilders.matchQuery("appkey", app)).must(QueryBuilders.existsQuery("errorCode")));
            }
            SearchResponse response = requestBuilder.execute().actionGet();
            return response.getHits().getTotalHits();
        }
        return 0;
    }

    /**
     * 获取调用错误数
     *
     * @return
     */
    public long getApiHourlyErrorCountByDate(Date date, String api) {
        DateFormat df = new SimpleDateFormat("yyyy-MM-dd");
        String index = df.format(date);
        boolean exists = elasticClient.admin().indices()
                .prepareExists("cloud_gw_log-" + index)
                .execute().actionGet().isExists();
        if (exists) {
            SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-" + df.format(date)).setTypes("api_log")
                    .setSearchType(SearchType.QUERY_AND_FETCH).setSize(0);
            requestBuilder.setQuery(QueryBuilders.existsQuery("errorCode")).setPostFilter(QueryBuilders.rangeQuery("@timestamp").from(date.getTime()).to(date.getTime() + 3600 * 1000 - 1));
            if (StringUtils.isNotEmpty(api)) {
                requestBuilder.setQuery(QueryBuilders.boolQuery().must(QueryBuilders.matchQuery("apiName", api)).must(QueryBuilders.existsQuery("errorCode")));
            }
            SearchResponse response = requestBuilder.execute().actionGet();
            return response.getHits().getTotalHits();
        }
        return 0;
    }
    /**
     * 获取调用错误数
     *
     * @return
     */
    public long getAppHourlyErrorCountByDate(Date date, String app) {
        DateFormat df = new SimpleDateFormat("yyyy-MM-dd");
        String index = df.format(date);
        boolean exists = elasticClient.admin().indices()
                .prepareExists("cloud_gw_log-" + index)
                .execute().actionGet().isExists();
        if (exists) {
            SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-" + df.format(date)).setTypes("api_log")
                    .setSearchType(SearchType.QUERY_AND_FETCH).setSize(0);
            requestBuilder.setQuery(QueryBuilders.existsQuery("errorCode")).setPostFilter(QueryBuilders.rangeQuery("@timestamp").from(date.getTime()).to(date.getTime() + 3600 * 1000 - 1));
            if (StringUtils.isNotEmpty(app)) {
                requestBuilder.setQuery(QueryBuilders.boolQuery().must(QueryBuilders.matchQuery("appkey", app)).must(QueryBuilders.existsQuery("errorCode")));
            }
            SearchResponse response = requestBuilder.execute().actionGet();
            return response.getHits().getTotalHits();
        }
        return 0;
    }

    public List<InvokeCount> getRealTimeInvokeCountList() {
        List<InvokeCount> list = new ArrayList<>();
        Calendar time = Calendar.getInstance();
        SearchRequestBuilder requestBuilder = elasticClient.prepareSearch("cloud_gw_log-*").setTypes("api_log")
                .setSearchType(SearchType.QUERY_AND_FETCH).addFields("@timestamp").setSize(0);
        requestBuilder.setQuery(QueryBuilders.rangeQuery("@timestamp").from(time.getTimeInMillis() - 3600 * 1000).to(time.getTime()));
        requestBuilder.addAggregation(AggregationBuilders.dateHistogram("count").field("@timestamp").interval(5 * 1000).timeZone("Asia/Shanghai").minDocCount(0).extendedBounds(time.getTimeInMillis() - (3600 * 1000), time.getTimeInMillis()));
        SearchResponse response = requestBuilder.execute().actionGet();
        InternalHistogram<InternalHistogram.Bucket> hourlyCounts = response.getAggregations().get("count");
        for (InternalHistogram.Bucket hourlyCount : hourlyCounts.getBuckets()) {
            DateTime date = (DateTime) hourlyCount.getKey();
            long millis = date.getMillis();
            long count = hourlyCount.getDocCount();
            InvokeCount invokeCount = new InvokeCount();
            invokeCount.setCount(count);
            invokeCount.setTime(millis);
            list.add(invokeCount);
        }
        return list;
    }
}

