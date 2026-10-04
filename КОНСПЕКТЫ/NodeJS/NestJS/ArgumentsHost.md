**ArgumentsHost** - это базовыйй объект которы инкапсулирует в себе аргументы передаваемые в обработчик вне зависимости от типа контекста 

```ts
export interface ArgumentsHost {
  getArgs<T extends Array<any> = any[]>(): T;
  getArgByIndex<T = any>(index: number): T;
  switchToHttp(): HttpArgumentsHost;
  switchToRpc(): RpcArgumentsHost;
  switchToWs(): WsArgumentsHost;
  getType<TContext extends string = ContextType>(): TContext;
}
```


Пример
```ts
@Catch()
export class HttpExecptionFilter implements ExceptionFilters {
	catch(exception: HttpException, host: ArgumentHost) {
		const ctx = host.switchToHttp()
		const response = ctx.getResponse<Response>()
		const request = ctx.getRequest<Request>()
		const status = exception.getStatus()
		
		response.status(status).json({
			statusCode: status,
			timestamp: new Date().now(),
			path: request.url,
			message: exception.message
		})
	}
}
```


```ts
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const contextType = host.getType();

    if (contextType === 'http') {
      const ctx = host.switchToHttp();
      const response = ctx.getResponse<Response>();
      response.status(500).json({ message: 'Internal error' });
    } else if (contextType === 'rpc') {
      const ctx = host.switchToRpc();
      // обработка ошибки для микросервиса
    } else if (contextType === 'ws') {
      const client = host.switchToWs().getClient();
      client.emit('error', { message: 'Internal error' });
    }
  }
}
```