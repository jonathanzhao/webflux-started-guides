# WebFlux项目
## 短链接
[创建短链接](http://su.yonyouup.com/su/create?url=https%3A%2F%2Fdocs.spring.io%2Fspring%2Fdocs%2Fcurrent%2Fspring-framework-reference%2Fweb-reactive.html%23webflux-fn)

> 路由和过滤器代码<br>
> 核心组件：RouterFunction / WebFilter / HandlerFilterFunction
```java
    @Bean
    public RouterFunction<ServerResponse> routerFunction(ShortUrlHandler handler) {
        return route()
                .GET("/su/create", handler::create)
                .GET("/favicon.ico", (request) -> status(HttpStatus.NOT_FOUND).build())
                .GET("/{code}", handler::route)
                .GET("/", handler::home)
//                .filter((request, next) ->
//                        next.handle(request).onErrorResume(ex -> {
//                            logger.error("未处理的异常 " + ex.getMessage(), ex);
//                            ResponseDto responseDto = new ResponseDto(false, ex.getMessage());
//                            return ok().contentType(MediaType.APPLICATION_JSON_UTF8).syncBody(responseDto);
//                        }).switchIfEmpty(Mono.defer(() -> ok().contentType(MediaType.APPLICATION_JSON_UTF8).syncBody(new ResponseDto(false))))
//                )
                .build();
    }

    @Bean
    public WebFilter webFilter() {
        return (ServerWebExchange exchange, WebFilterChain chain) ->
                chain.filter(exchange).onErrorResume(ex -> {
                    logger.error("未处理的异常 " + ex.getMessage(), ex);
                    ResponseDto responseDto = new ResponseDto(false, ex.getMessage());
                    try {
                        exchange.getResponse().getHeaders().add("Content-Type", MediaType.APPLICATION_JSON_UTF8_VALUE);
                        byte[] data = jsonWrapper.getMapper().writeValueAsBytes(responseDto);
                        DataBuffer buffer = exchange.getResponse().bufferFactory().wrap(data);
                        return exchange.getResponse().writeWith(Mono.just(buffer));
                    } catch (JsonProcessingException e) {
                        return Mono.empty();
                    }
                });
    }
```
> 重定向代码<br/>
> 核心组件：HandlerFunction
```java
    public Mono<ServerResponse> route(ServerRequest request) {
        String shortCode = request.pathVariable("code");
        if (shortCode == null || shortCode.length() < 8) {
            Map<String, String> model = new HashMap<>();
            model.put("msg", "无效的请求");
            return ok().contentType(TEXT_HTML_UTF8).render("error", model);
        } else {
            return urlRepository.get(shortCode).defaultIfEmpty("").flatMap(sourceUrl -> {
                if (StringUtils.isEmpty(sourceUrl)) {
                    Map<String, String> model = new HashMap<>();
                    model.put("msg", "短链接不存在");
                    return ok().contentType(TEXT_HTML_UTF8).render("error", model);
                } else {
                    return temporaryRedirect(URI.create(sourceUrl)).build();
                }
            });
        }
    }
```

## 快递路由
[快递单号查询](http://express.yonyouup.com/)

快递单号
- [ ] TT9900001766878
- [ ] 805281569547522089
- [ ] 3704599920965
<!-- - [ ] 75140426200846 -->
<!-- - [ ] 336043783895 -->
<!-- - [ ] 9895177479778 -->
<!-- - [ ] 3708760385212 -->

> 响应式非阻塞HTTP请求代码<br/>
> 核心组件：WebClient
```java
    protected <T, R> Mono<T> get(String url, Map<String, String> headers, ClientRequestConfig requestConfig, Function<R, T> converter, Class<R> clazz) {
        Mono<T> result = createRequestHeadersSpec(url, headers)
                .retrieve()
                .bodyToMono(clazz)
                .map(s -> converter.apply(s));
        return result;
    }

    protected <T> Mono<T> formPost(String url, String content, Function<String, T> converter) {
        Mono<T> result = createRequestBodySpec(url, null)
                .contentType(MediaType.APPLICATION_FORM_URLENCODED)
                .body(fromObject(content))
                .retrieve()
                .bodyToMono(String.class).map(s -> converter.apply(s));
        return result;
    }

    protected <T, R> Mono<T> post(String url, String content, Map<String, String> headers, ClientRequestConfig requestConfig, Function<R, T> converter, Class<R> clazz) {
        Mono<T> result = createRequestBodySpec(url, headers)
                .body(fromObject(content))
                .retrieve()
                .bodyToMono(clazz).map(s -> converter.apply(s));
        return result;
    }

    protected Mono<String> cookie(String url, Map<String, String> headers, String header) {
        Mono<String> result = createRequestBodySpec(url, headers)
                .exchange()
                .map(response -> response.headers().header(header).stream().map(h -> {
                    int i = h.indexOf(";");
                    if (i > 0) {
                        h = h.substring(0, i + 1);
                    }
                    return h;
                }).collect(joining("; ")));
        return result;
    }

    protected WebClient.RequestHeadersSpec createRequestHeadersSpec(String url, Map<String, String> headers) {
        WebClient.RequestHeadersSpec spec = webClient.get().uri(url);
        if (headers != null && headers.size() > 0) {
            headers.entrySet().forEach(entry -> spec.header(entry.getKey(), entry.getValue()));
        }
        return spec;
    }

    protected WebClient.RequestBodySpec createRequestBodySpec(String url, Map<String, String> headers) {
        WebClient.RequestBodySpec spec = webClient.post().uri(url);
        if (headers != null && headers.size() > 0) {
            headers.entrySet().forEach(entry -> spec.header(entry.getKey(), entry.getValue()));
        }
        return spec;
    }
```

## 看板
[营销云看板](http://yxykb.yonyouup.com/)

> 为HandlerFunction传递Context参数<br/>
> 核心组件：RouterFunction
```java
    @Bean
    public RouterFunction<ServerResponse> routerFunction() {
        return route()
                .path("/user", builder -> builder
                        .GET("/name", buildHandlerFunction(userHandler::name))
                        .POST("/resetpwd", buildHandlerFunction(userHandler::resetPassword))
                        .POST("/save", buildHandlerFunction(userHandler::save))
                        .POST("/login", buildHandlerFunction(loginHandler::login))
                        .GET("/logout", buildHandlerFunction(loginHandler::logout))
                )
                .path("/query", builder -> builder
                        .GET("/group/{name}", buildHandlerFunction(localSchemaHandler::groupQuery))
                        .GET("/{name}", buildHandlerFunction(localSchemaHandler::query))
                        .GET("/group/format/{name}", buildHandlerFunction(localSchemaHandler::groupQueryFormat))
                        .GET("/format/{name}", buildHandlerFunction(localSchemaHandler::queryFormat))
                        .GET("/template/reload", localSchemaHandler::reloadTemplate)
                )
                .path("/rest", builder -> builder
                        .POST("/union/query", buildHandlerFunction(queryHandler::groupQuery))
                        .POST("/{m}/{c}/{e}/query", buildHandlerFunction(queryHandler::query))
                        .POST("/{m}/{c}/{e}/count", buildHandlerFunction(queryHandler::count))
                )
                .path("/io", builder -> builder
                        .POST("/{m}/{c}/{e}/export", buildHandlerFunction(fnHandler::export))
                )
                .GET("/ext/group/industry", buildHandlerFunction(localSchemaHandler::industry))
                .GET("/favicon.ico", (request) -> status(HttpStatus.NOT_FOUND).build())
                .GET("/", loginHandler::home)
                .filter(kanbanFilter)
                .build();
    }

    // 从Context中获取自定义参数ServiceContext传递给HandlerFunction
    private HandlerFunction<ServerResponse> buildHandlerFunction(BiFunction<ServerRequest, ServiceContext, Mono<ServerResponse>> handlerFunction) {
        return request -> Mono.subscriberContext().flatMap(ctx ->
                handlerFunction.apply(request, ctx.get("service-context"))
        );
    }
```

> 权限校验与Context设置<br />
> 核心组件：HandlerFilterFunction / Context
```java
    @Override
    public Mono<ServerResponse> filter(ServerRequest request, HandlerFunction<ServerResponse> next) {
        ServiceContext serviceContext = ServiceContextBuilder.createServiceContext();
        setEnv(request, serviceContext);
        return authCheck(request).flatMap(b -> next.handle(request)
                .switchIfEmpty(Mono.defer(() -> ok().syncBody(new ResponseDto(false, "未找到数据").setStatus(HttpStatus.BAD_REQUEST))))
                .subscriberContext(Context.of("service-context", serviceContext))
        ).onErrorResume(ex -> {
            ex = Exceptions.unwrap(ex);
            if (ex instanceof NeedLoginException || ex instanceof AuthException) {
                ResponseDto responseDto = new ResponseDto(false, ex.getMessage()).setStatus(HttpStatus.FOUND);
                return ok().contentType(MediaType.APPLICATION_JSON_UTF8).syncBody(responseDto);
            } else {
                ResponseDto responseDto = new ResponseDto(false, ex.getMessage()).setStatus(HttpStatus.BAD_REQUEST);
                return ok().contentType(MediaType.APPLICATION_JSON_UTF8).syncBody(responseDto);
            }
        });
    }

    // 权限校验，不能直接抛出异常，使用Mono.error()包装
    private Mono<Boolean> authCheck(ServerRequest request) {
        try {
            String path = request.path();
            if (path.length() == 1 || path.indexOf('.') > 0) {
                return Mono.just(true);
            }
            boolean isAuthentication = path.indexOf("/login") < 0;
            if (!isAuthentication) {
                return Mono.just(true);
            }
            String authHeader = WebUtils.getHeader(request, kanbanConfig.getAuthHeaderKey());
            if (!StringUtils.isEmpty(authHeader)) { // api 授权
                String[] arr = authHeader.split(" ");
                if (arr.length != 3 ||
                        StringUtils.isEmpty(arr[0]) ||
                        StringUtils.isEmpty(arr[1]) ||
                        StringUtils.isEmpty(arr[2]) ||
                        !instantPattern.matcher(arr[1]).find()) {
                    throw Exceptions.propagate(new AuthException("API AUTHENTICATION头格式不合法"));
                }
                return cacheRepository.getAppSecret(arr[0]).map(appSecret -> {
                    if (StringUtils.isEmpty(appSecret)) {
                        throw Exceptions.propagate(new AuthException("API 非法的AppKey"));
                    }
                    if (!BooleanUtils.b(kanbanConfig.getApiSkipCheck())) {
                        if (Math.abs(Instant.now().toEpochMilli() - Long.valueOf(arr[1])) > kanbanConfig.getApiToleranceMilli()) {
                            throw Exceptions.propagate(new AuthException("API 时间戳已经超期"));
                        }
                    }
                    request.attributes().put(kanbanConfig.getAppSecrete(), appSecret);
                    return true;
                });
            } else { // 登录校验
                String token = getToken(request);
                //如果Token不存在，需要登录
                if (StringUtils.isEmpty(token)) {
                    throw Exceptions.propagate(new NeedLoginException("请登录后再继续"));
                }
                //构造登录用户，如果已经失效，需要登录
                return cacheRepository.userExists(token).map(b -> {
                    if (!b) {
                        throw Exceptions.propagate(new NeedLoginException("登录已过期，请重新登录"));
                    }
                    if (BooleanUtils.b(kanbanConfig.getAuthentication().getSliding())) {
                        cacheRepository.sliding(token, kanbanConfig.getAuthentication().getExpires());
                    }
                    request.attributes().put(kanbanConfig.getTokenKey(), token);
                    return true;
                });
            }
        } catch (Throwable ex) {
            return Mono.error(ex);
        }
    }
```

> 调用RPC服务<br/>
> 核心组件：HandlerFunction / Dubbo
```java
    public Mono<ServerResponse> groupQuery(ServerRequest request, ServiceContext serviceContext) {
        return body(request).flatMap(schemaJson -> {
            Map<String, Map<String, Object>> results = null;
            try {
                results = kanbanQueryService.groupQuery(schemaJson, createServiceContext());
            } catch (BusinessException | ServiceException e) {
                throw Exceptions.propagate(e);
            }
            ResponseDto responseDto = new ResponseDto(true).setData(results);
            if (results == null || results.size() == 0) {
                responseDto.setMessage("未查询到数据");
            }
            return ok().contentType(MediaType.APPLICATION_JSON_UTF8).syncBody(responseDto);
        });
    }
```

> 导出Excel代码<br/>
> 核心组件：MDD / HandlerFunction / DataBuffer
```java
    public Mono<ServerResponse> export(ServerRequest request, ServiceContext serviceContext) {
        String m = request.pathVariable("m");
        String c = request.pathVariable("c");
        String e = request.pathVariable("e");
        String fullname = String.format("%s.%s.%s", m, c, e);
        return request.formData().flatMap(form -> {
            try {
                String type = form.getFirst("type");
                String name = form.getFirst("name");
                String exportConfigName = null;
                if (StringUtils.isEmpty(type)) {
                    exportConfigName = String.format("%s-%s-%s", m, c, e);
                } else {
                    exportConfigName = String.format("%s-%s-%s-%s", m, c, e, type);
                }
                exportConfigName = exportConfigName.toLowerCase();
                List<ExportTableConfig> configs = exportConfig.getExportTableConfig(exportConfigName);
                if (configs == null) {
                    Map<String, String> model = new HashMap<>();
                    model.put("msg", "没有配置文件");
                    return ok().contentType(WebUtils.TEXT_HTML_UTF8).render("download-error", model);
                }
                String schemeJson = form.getFirst("data");
                //data
                QueryResult result = loadData(fullname, schemeJson);
                if (result == null || result.getCount() == 0) {
                    throw new BusinessException("没有需要导出的数据");
                }
                DataSet ds = toDataSet(result, e);
                String exportFileName = name;
                if (StringUtils.isEmpty(exportFileName)) {
                    exportFileName = String.format("%s-%s.xlsx", e, DateTimeUtils.minTimeOfToday());
                } else {
                    exportFileName = exportFileName + ".xlsx";
                }
                //excel template
                String tplFile = FilePathUtils.combine(exportConfig.getBaseDir(), exportConfigName + ExportConfig.excelFileSuffix);
                ExcelTableWriter writer = new ExcelTableWriter();
//                ByteArrayOutputStream baos = new ByteArrayOutputStream();
                DataBuffer dataBuffer = new DefaultDataBufferFactory().allocateBuffer();
                writer.write(new File(tplFile), ds, configs, dataBuffer.asOutputStream());
                return ok().contentType(WebUtils.APP_EXCEL_UTF8)
                        .header("Content-disposition", "attachment;filename=\"" + new String(exportFileName.getBytes(), "ISO8859-1") + "\"")
                        .body(Mono.just(dataBuffer), DataBuffer.class);
                //.syncBody(baos.toByteArray());
            } catch (Exception ex) {
                Map<String, String> model = new HashMap<>();
                model.put("msg", ex.getMessage());
                return ok().contentType(WebUtils.TEXT_HTML_UTF8).render("download-error", model);
            }
        });
    }
```

> 聚合查询<br/>
> 核心组件：MDD
```java
    public Mono<ServerResponse> groupQueryFormat(ServerRequest request, ServiceContext context) {
        String name = request.pathVariable("name");
        String json = groupQueryFormat(name, context);
        return ok().contentType(MediaType.APPLICATION_JSON_UTF8).syncBody(json);
    }

    // 将聚合查询出来的结果通过模版组装数据
    private String groupQueryFormat(String name, ServiceContext context) {
        String json = null;
        try {
            String schemaJson = loadSchema(name);
            Map<String, Map<String, Object>> data = kanbanQueryService.groupQuery(schemaJson, createServiceContext());
            if (data == null || data.size() == 0) {
                json = renderSuccess("未查找到数据");
            } else {
                TplContext tplContext = tplBeanFactory.getTplContext(MapValueHandler.class);
                tplContext.setAppendNull(true);
                TplDataSource<Map<String, Map<String, Object>>> tplDs = new TplDataSource<>(data);
                tplDs.set("JSON_TPL_KEY", name + "-result.tpl");
                TplBuilder builder = tplContext.getTplBuilder("JsonBuilder");
                StringBuilder sb = builder.render(tplContext, tplDs);
                json = sb == null ? "" : sb.toString();
            }
        } catch (Throwable e) {
            logger.error("批量查询并格式化失败", e);
            json = renderError(e.getMessage());
        }
        return json;
    }
```

> 查询方案
```json
[
  {
    "name": "monthProductQty",
    "fullname": "kb.stat.ProductActive",
    "fields": [
      {"name": "productQty","alias": "monthProductQty","aggr": "sum"},
      {"name": "concat(statTime.year,'-',concat(if(statTime.month<10) then '0' else '' end,statTime.month),'月')","alias": "yearMouth"}
    ],
    "conditions": [
      {"name": "statTime","op": "between","v1": "current_date-11m/min_m","v2": "current_date/max"}
    ],
    "groups": [
      {"name": "statTime.year"},
      {"name": "statTime.month"}
    ],
    "orders": [
      {"name": "statTime.year"},
      {"name": "statTime.month"}
    ]
  },
  {
    "name": "totalProductQty",
    "fullname": "kb.stat.ProductActive",
    "fields": [
      {"name": "productQty","alias": "totalProductQty","aggr": "sum"}
    ]
  },
  {
    "name": "newProductQty",
    "fullname": "kb.stat.ProductActive",
    "fields": [
      {"name": "productQty","alias": "newProductQty","aggr": "sum"}
    ],
    "conditions": [
      {"name": "statTime","op": "between","v1": "current_date-30d","v2": "current_date/max"}
    ]
  },
  {
    "name": "productSummary",
    "fullname": "kb.stat.ProductSummary",
    "fields": [
      {"name": "productClassQty","alias": "totalProductClassQty","aggr": "sum"}
    ]
  },
  {
    "name": "2bUdhTotal",
    "fullname": "kb.stat.OrderActive",
    "fields": [
      {"name": "orderQty","alias": "totalOrderQty","aggr": "sum"},
      {"name": "orderMoney","alias": "totalOrderMoney","aggr": "sum"}
    ]
  },
  {
    "name": "2bUdhNew",
    "fullname": "kb.stat.OrderActive",
    "fields": [
      {"name": "orderQty","alias": "newOrderQty","aggr": "sum"},
      {"name": "orderMoney","alias": "newOrderMoney","aggr": "sum"}
    ],
    "conditions": [
      {"name": "statTime","op": "between","v1": "current_date-30d","v2": "current_date/max"}
    ]
  },
  {
    "name": "2cMallTotal",
    "fullname": "kb.stat.MallActive",
    "fields": [
      {"name": "orderQty","alias": "totalOrderQty","aggr": "sum"},
      {"name": "orderMoney","alias": "totalOrderMoney","aggr": "sum"}
    ]
  },
  {
    "name": "2cMallNew",
    "fullname": "kb.stat.MallActive",
    "fields": [
      {"name": "orderQty","alias": "newOrderQty","aggr": "sum"},
      {"name": "orderMoney","alias": "newOrderMoney","aggr": "sum"}
    ],
    "conditions": [
      {"name": "statTime","op": "between","v1": "current_date-30d","v2": "current_date/max"}
    ]
  },
  {
    "name": "2cRetailTotal",
    "fullname": "kb.stat.RetailActive",
    "fields": [
      {"name": "orderQty","alias": "totalOrderQty","aggr": "sum"},
      {"name": "orderMoney","alias": "totalOrderMoney","aggr": "sum"}
    ]
  },
  {
    "name": "2cRetailNew",
    "fullname": "kb.stat.RetailActive",
    "fields": [
      {"name": "orderQty","alias": "newOrderQty","aggr": "sum"},
      {"name": "orderMoney","alias": "newOrderMoney","aggr": "sum"}
    ],
    "conditions": [
      {"name": "statTime","op": "between","v1": "current_date-30d","v2": "current_date/max"}
    ]
  },
  {
    "name": "newMemberQty",
    "fullname": "kb.stat.MemberActive",
    "fields": [
      {"name": "newMemQty","alias": "newMemberQty","aggr": "sum"}
    ],
    "conditions": [
      {"name": "statTime","op": "between","v1": "current_date-30d","v2": "current_date/max"}
    ]
  },
  {
    "name": "totalMemberQty",
    "fullname": "kb.stat.MemberActive",
    "fields": [
      {"name": "newMemQty","alias": "totalMemberQty","aggr": "sum"}
    ]
  }
]
```

> 格式化模版
```xhtml
{
    "success": true,
    "msg": "success",
    "status": 200,
    "data": {
        "goodsTotalCount": <#=totalProductQty.data[0].totalProductQty#>,
        "goodsMonthCount": <#=newProductQty.data[0].newProductQty#>,
        "goodsClassTotalCount": <#=productSummary.data[0].totalProductClassQty#>,
        "tradeTotalCountToB": <#=2bUdhTotal.data[0].totalOrderQty#>,
        "tradeMonthCountToB": <#=2bUdhNew.data[0].newOrderQty#>,
        "tradeTotalCountToC": <#/iadd(2cMallTotal.data[0].totalOrderQty, 2cRetailTotal.data[0].totalOrderQty)#>,
        "tradeMonthCountToC": <#/iadd(2cMallNew.data[0].newOrderQty, 2cRetailNew.data[0].newOrderQty)#>,
        "tradeTotalMoneyToB": <#=2bUdhTotal.data[0].totalOrderMoney#>,
        "tradeMonthMoneyToB": <#=2bUdhNew.data[0].newOrderMoney#>,
        "tradeTotalMoneyToC": <#/add(2cMallTotal.data[0].totalOrderMoney, 2cRetailTotal.data[0].totalOrderMoney)#>,
        "tradeMonthMoneyToC": <#/add(2cMallNew.data[0].newOrderMoney, 2cRetailNew.data[0].newOrderMoney)#>,
        "memberMonthCount": <#=newMemberQty.data[0].newMemberQty#>,
        "memberTotalCount": <#=totalMemberQty.data[0].totalMemberQty#>,
        "goodsGrowth": {
            "xAxis": {
                "data": [
                <#?(monthProductQty.totalCount>0){#>
                <#*(d:monthProductQty.data){#>
<#=monthProductQty#><#?(IS_LAST!=true)#>,<#}#>
                <#}#>

                <#}#>
                ]
            },
            "series": [
                {
                    "data": [
                        <#?(monthProductQty.totalCount>0){#>
                        <#*(d:monthProductQty.data){#>
"<#=yearMouth#>"<#?(IS_LAST!=true)#>,<#}#>
                        <#}#>

                        <#}#>
                    ],
                    "name": "商品增长量"
                }
            ],
            "name": "商品增长量"
        }
    }
}
```

> 聚合查询<br/>
> URL：http://yxykb.yonyouup.com/query/group/industry
```json
{
    "flag": 1,
    "success": true,
    "msg": "查询成功",
    "status": 200,
    "totalCount": "12",
    "data": {
        "2bUdhTotal": {
            "flag": 1,
            "data": [
                {
                    "totalOrderQty": 4010542,
                    "totalOrderMoney": 55229442657.15
                }
            ],
            "totalCount": 1,
            "status": 200
        },
        "totalProductQty": {
            "flag": 1,
            "data": [
                {
                    "totalProductQty": 158402
                }
            ],
            "totalCount": 1,
            "status": 200
        },
        "2cMallTotal": {
            "flag": 1,
            "data": [
                {
                    "totalOrderQty": 508850,
                    "totalOrderMoney": 459948772.6
                }
            ],
            "totalCount": 1,
            "status": 200
        },
        "newProductQty": {
            "flag": 1,
            "data": [
                {
                    "newProductQty": 20151
                }
            ],
            "totalCount": 1,
            "status": 200
        },
        "2cMallNew": {
            "flag": 1,
            "data": [
                {
                    "newOrderQty": 21007,
                    "newOrderMoney": 12658603.76
                }
            ],
            "totalCount": 1,
            "status": 200
        },
        "2cRetailTotal": {
            "flag": 1,
            "data": [
                {
                    "totalOrderQty": 358682,
                    "totalOrderMoney": 1042507645.01
                }
            ],
            "totalCount": 1,
            "status": 200
        },
        "newMemberQty": {
            "flag": 1,
            "data": [
                {
                    "newMemberQty": 132697
                }
            ],
            "totalCount": 1,
            "status": 200
        },
        "2cRetailNew": {
            "flag": 1,
            "data": [
                {
                    "newOrderQty": 75880,
                    "newOrderMoney": 21222540.01
                }
            ],
            "totalCount": 1,
            "status": 200
        },
        "productSummary": {
            "flag": 1,
            "data": [
                {
                    "totalProductClassQty": 12685
                }
            ],
            "totalCount": 1,
            "status": 200
        },
        "2bUdhNew": {
            "flag": 1,
            "data": [
                {
                    "newOrderQty": 273547,
                    "newOrderMoney": 2751528332.56
                }
            ],
            "totalCount": 1,
            "status": 200
        },
        "totalMemberQty": {
            "flag": 1,
            "data": [
                {
                    "totalMemberQty": 4106909
                }
            ],
            "totalCount": 1,
            "status": 200
        },
        "monthProductQty": {
            "flag": 1,
            "data": [
                {
                    "monthProductQty": 1884,
                    "yearMouth": "2018-05月"
                },
                {
                    "monthProductQty": 18353,
                    "yearMouth": "2018-06月"
                },
                {
                    "monthProductQty": 5249,
                    "yearMouth": "2018-07月"
                },
                {
                    "monthProductQty": 9872,
                    "yearMouth": "2018-08月"
                },
                {
                    "monthProductQty": 11997,
                    "yearMouth": "2018-09月"
                },
                {
                    "monthProductQty": 6800,
                    "yearMouth": "2018-10月"
                },
                {
                    "monthProductQty": 7781,
                    "yearMouth": "2018-11月"
                },
                {
                    "monthProductQty": 21391,
                    "yearMouth": "2018-12月"
                },
                {
                    "monthProductQty": 5032,
                    "yearMouth": "2019-01月"
                },
                {
                    "monthProductQty": 7250,
                    "yearMouth": "2019-02月"
                },
                {
                    "monthProductQty": 16404,
                    "yearMouth": "2019-03月"
                },
                {
                    "monthProductQty": 7150,
                    "yearMouth": "2019-04月"
                }
            ],
            "totalCount": 12,
            "status": 200
        }
    }
}
```

> 聚合查询并格式化数据<br/>
> URL：http://yxykb.yonyouup.com/query/group/format/industry
```json
{
    "success": true,
    "msg": "success",
    "status": 200,
    "data": {
        "goodsTotalCount": 158402,
        "goodsMonthCount": 20151,
        "goodsClassTotalCount": 12685,
        "tradeTotalCountToB": 4010542,
        "tradeMonthCountToB": 273547,
        "tradeTotalCountToC": 867532,
        "tradeMonthCountToC": 96887,
        "tradeTotalMoneyToB": 55229442657.15,
        "tradeMonthMoneyToB": 2751528332.56,
        "tradeTotalMoneyToC": 1502456417.61,
        "tradeMonthMoneyToC": 33881143.77,
        "memberMonthCount": 132697,
        "memberTotalCount": 4106909,
        "goodsGrowth": {
            "xAxis": {
                "data": [
                    1884,
                    18353,
                    5249,
                    9872,
                    11997,
                    6800,
                    7781,
                    21391,
                    5032,
                    7250,
                    16404,
                    7150
                ]
            },
            "series": [
                {
                    "data": [
                        "2018-05月",
                        "2018-06月",
                        "2018-07月",
                        "2018-08月",
                        "2018-09月",
                        "2018-10月",
                        "2018-11月",
                        "2018-12月",
                        "2019-01月",
                        "2019-02月",
                        "2019-03月",
                        "2019-04月"
                    ],
                    "name": "商品增长量"
                }
            ],
            "name": "商品增长量"
        }
    }
}
```
