# NestJS에서 Socket 게이트웨이의 CORS 설정

`main.ts`에 설정한 CORS 외에도 별도로 설정이 필요함

```typescript
@WebSocketGateway({
    cors: {
        origin: [
            `https://example.com`, // Frontend URL
        ],
        credentials: true,
    },
})
export class TimerGateway
    implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect {
    @WebSocketServer() server: Server;

    constructor() {}

    afterInit(server: Server) {
        if (process.env.NODE_ENV === 'development') {
            console.log('WebSocket Initialized');
            console.log(server._opts);
        }
    }
    
//...

}
```