# ES中删除指定记录中的一个字段
## 需求描述
1. 项目中的所有客户信息都保存在ES中，由于以前的接口调用链太深，修改起来，你懂的，所以决定重新写一个删除字段的方法
2. 方法尽量通用

## 实现
1. 依赖
```
        <dependency>  
            <groupId>dx.commons</groupId>
			<artifactId>dx-commons-elasticsearch</artifactId>
			<version>1.0.0</version>
		</dependency>
	    <dependency>
			<groupId>org.elasticsearch</groupId>
			<artifactId>elasticsearch</artifactId>
			<version>5.6.3</version>
		</dependency>
		<dependency>
			<groupId>org.elasticsearch.client</groupId>
			<artifactId>transport</artifactId>
			<version>5.6.3</version>
		</dependency>
		<dependency>
			<groupId>org.elasticsearch.client</groupId>
			<artifactId>elasticsearch-rest-high-level-client</artifactId>
			<version>5.6.3</version>
		</dependency>```


2. JSON
```
    {
    "took": 2,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 1,
        "max_score": 1,
        "hits": [
            {
                "_index": "asst_customers_v1",
                "_type": "default",
                "_id": "1000212",
                "_score": 1,
                "_source": {
                    "createTime": 1513059302000,
                    "updateTime": 1513058943000,
                    "customerGroupId": 6,
                    "crmCustomerId": ccc,
                    "sourcePlatform": "schoki",
                    "custGroupId": -6,
                    "followWechatOfficialAccountFlag": true
                }
            }
        ]
    }
}   ```




2.  实现类
```
        package com.dx.asst.customer.service;
        import java.io.IOException;
        import org.apache.http.util.Asserts;
        import org.elasticsearch.action.update.UpdateRequest;
        import org.elasticsearch.client.RestHighLevelClient;
        import org.elasticsearch.script.Script;
        import org.springframework.beans.factory.annotation.Autowired;
        import org.springframework.stereotype.Service;
        import com.dx.asst.comm.constants.EsIndexConstants;
        import lombok.extern.slf4j.Slf4j;
        /**
        * @description :
        * @author : zhubing.ji
        * @date : 2018/6/14 下午4:44
        */
        @Slf4j
        @Service
        public class CustomerOperationEsServiceTest {
        private static final String DEFAULT_TYPE = "default";
        @Autowired
        private RestHighLevelClient restHighLevelClient;
        //说明，这个remove里面如果没有单引号的话，会报错
        private final String updateCustomerScript = "ctx._source.remove('%s')";

         /**
            * 
            * @param customerId 客户记录中的ID
            * @param keys 要删除的字段集合
            */
        
        public void deleteByAttrs(Long customerId, Iterable<String> keys) {
            Asserts.notNull(customerId, "customerId is null");
            Asserts.notNull(keys, "keys is null");

        UpdateRequest updateRequest = new UpdateRequest(EsIndexConstants.CUSTOMER, "default", customerId.toString());
            
            keys.forEach(key -> {
                String updateScript = String.format(updateCustomerScript, key);
                updateRequest.script(new Script(updateScript));
                try {
                    restHighLevelClient.update(updateRequest);
                } catch (IOException e) {
                    log.error("when delete customer attr error", e);
                    throw new RuntimeException("when releaseBindUser error", e);
                }
            });
        }
        }

```

4. 说明
        如果想删除JSON中的custGroupId和crmCustomerId，调用方法的参数只要按照以下方式传递就可以了
        
        ```
        List<String> keys = Lists.newArrayListWithCapacity(2); 
		keys.add("bindUser");
		keys.add("crmCustomerId");
		customerClientService.deleteAttrsById(customerId, keys);
        ```
