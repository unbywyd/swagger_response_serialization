# Express + Swagger + Response Serialization

### Used packages:
* [express](https://www.npmjs.com/package/express)
* [swaggerTools](https://www.npmjs.com/package/swagger-tools)
* [json-refs](https://www.npmjs.com/package/json-refs) (Not required)
* [fast-json-stringify](https://www.npmjs.com/package/fast-json-stringify)

```js
const app = express();
....
app.use(express.json());

JsonRefs.resolveRefsAt('./app/apidocs/swagger.json').then(results => {
        
    /**
    * Swagger tools initialization:
    * Swagger-ui, swagger-validator, swagger-router & swagger-security
    */
    swaggerTools.initializeMiddleware(results.resolved, function (middleware) {

        app.use(middleware.swaggerMetadata());
        
        /**
        * Register our method that will be responsible for serialization
        */
        app.use(function (req, res, next) {
            let method = req.method.toLowerCase();

            /**
            * Custom method that will return serialized data relative to the response schema    
            */
            res.JSON = function(data) {
                if(req.swagger.path[method]) {
                    let responseSchemes = req.swagger.path[method].responses;
                    if(responseSchemes && responseSchemes[res.statusCode]) {
                        let responseScheme = responseSchemes[res.statusCode].schema;
                        if(responseScheme) {                        
                            let stringify = fastJson(responseScheme);
                            res.setHeader('Content-Type', 'application/json');
                            return res.end(stringify(data));
                        }
                    }
                }                    
                return res.json(data);                
            }  
            next();
        });

        app.use(middleware.swaggerValidator());
        app.use(middleware.swaggerUi());
        
        app.use(
            middleware.swaggerSecurity({})
        );

        /**
         * Init custom routing
         * before swagger routing.
         */
        app.use(require('./routes').router);
        
        app.use(middleware.swaggerRouter({
            controllers: './app/controllers',
            useStubs: false
        }));

        app.use(apiErrorHandler);

    });
});
```

## Usage

### Swagger Scheme

```json
{
    "swagger": "2.0",
    ...
    "paths": {
          "/wordpress/plugin": {
            "get": {
                "x-swagger-router-controller": "wordpress",
                "operationId": "getPlugin",
                "security": [],
                "summary": "Get plugin",
                "responses": {
                    "200": {
                        "description": "Plugin details",
                        "schema": {
                            "type": "object",
                            "additionalProperties": false,
                            "properties": {
                                "id": {
                                    "type": "string"
                                },
                                "name": {
                                    "type": "string"
                                },
                                "version": {
                                    "type": "number"  
                                },
                                "active": {
                                    "type": "boolean"
                                },
                                "slug": {
                                    "type": "string"
                                },
                                "url": {
                                    "type": "string"
                                },
                                "created_at": {
                                    "type": "string"
                                },
                                "updated_at": {
                                    "type": "string"
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
```

### getPlugin route handler

```js
exports.getPlugin = wrap(async (req, res, next) => {
    res.JSON({
        name: 'Test',
        _id: 'MongoDB ID',
        id: 123,
        version: '1',
        __v: '1.0',
        other_field: 'value',
        slug: 'test'
    });
});
```

and then the response will be:

```json
{
    "name": "Test",
    "id": "123",
    "version": 1,
    "slug": "test"
}

```

