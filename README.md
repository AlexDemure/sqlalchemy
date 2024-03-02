# Example SqlAlchemy with PostgreSQL
### ```Setup.py```
```python
import json
from contextlib import asynccontextmanager
from functools import partial

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.ext.asyncio import async_sessionmaker
from sqlalchemy.ext.asyncio import create_async_engine
from sqlalchemy.ext.declarative import declarative_base


engine = create_async_engine(
    url,
    future=True,
    isolation_level="READ COMMITTED",
)

sessionmaker = async_sessionmaker(_engine, expire_on_commit=False)

Table = declarative_base()


@asynccontextmanager
async def read() -> AsyncSession:
    async with sessionmaker() as session:
        # await sesion.begin()
        await session.connection(execution_options={"isolation_level": "AUTOCOMMIT"}) - # disable send commands to Postgres [BEGIN, ROLLBACK] through session.begin(), session.rollback() 
        yield session
        # await sesion.rollback()
        # await session.close() - clear connection

@asynccontextmanager
async def write() -> AsyncSession:
    async with sessionmaker() as session:
        try:
            async with session.begin():
                yield session
                # session.commit() 
        except Exception as e:
            await session.rollback()
            raise e
        # await session.close() - clear connection

@asynccontextmanager
async def pgconnect(transaction: bool = False) -> AsyncSession:
    if transaction:
        async with write() as session:
            yield session
    else:
        async with read() as session:
            yield session
```
### Usage
```python
import asyncio

from sqlalchemy import text

async def main():
    async with pgconnect() as session:
        await session.execute(text('select 1'))

    async with pgconnect(transaction=True) as session:
        await session.execute(text('select 2'))


if __name__ == '__main__':
    asyncio.run(main())
```
### Postgres Logs
```
2024-03-02 07:49:13.473 UTC [37] LOG:  execute __asyncpg_stmt_7__: select 1
2024-03-02 07:49:13.474 UTC [37] LOG:  statement: BEGIN ISOLATION LEVEL READ COMMITTED;
2024-03-02 07:49:13.474 UTC [37] LOG:  execute __asyncpg_stmt_8__: select 2
2024-03-02 07:49:13.474 UTC [37] LOG:  statement: COMMIT;

```
### App logs
```
2024-03-02 12:49:13,467 DEBUG sqlalchemy.pool.impl.AsyncAdaptedQueuePool Created new connection <AdaptedConnection <asyncpg.connection.Connection object at 0x7f3d87c88a90>>
2024-03-02 12:49:13,472 DEBUG sqlalchemy.pool.impl.AsyncAdaptedQueuePool Connection <AdaptedConnection <asyncpg.connection.Connection object at 0x7f3d87c88a90>> checked out from pool
>> 2024-03-02 12:49:13,472 INFO sqlalchemy.engine.Engine BEGIN (implicit; DBAPI should not BEGIN due to autocommit mode)
2024-03-02 12:49:13,472 INFO sqlalchemy.engine.Engine select 1
2024-03-02 12:49:13,473 INFO sqlalchemy.engine.Engine [generated in 0.00008s] ()
>> 2024-03-02 12:49:13,473 INFO sqlalchemy.engine.Engine ROLLBACK using DBAPI connection.rollback(), DBAPI should ignore due to autocommit mode
2024-03-02 12:49:13,473 DEBUG sqlalchemy.pool.impl.AsyncAdaptedQueuePool Connection <AdaptedConnection <asyncpg.connection.Connection object at 0x7f3d87c88a90>> being returned to pool
2024-03-02 12:49:13,473 DEBUG sqlalchemy.pool.impl.AsyncAdaptedQueuePool Connection <AdaptedConnection <asyncpg.connection.Connection object at 0x7f3d87c88a90>> rollback-on-return
2024-03-02 12:49:13,473 DEBUG sqlalchemy.pool.impl.AsyncAdaptedQueuePool Connection <AdaptedConnection <asyncpg.connection.Connection object at 0x7f3d87c88a90>> checked out from pool
>> 2024-03-02 12:49:13,473 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2024-03-02 12:49:13,473 INFO sqlalchemy.engine.Engine select 2
2024-03-02 12:49:13,474 INFO sqlalchemy.engine.Engine [generated in 0.00006s] ()
>> 2024-03-02 12:49:13,474 INFO sqlalchemy.engine.Engine COMMIT
2024-03-02 12:49:13,475 DEBUG sqlalchemy.pool.impl.AsyncAdaptedQueuePool Connection <AdaptedConnection <asyncpg.connection.Connection object at 0x7f3d87c88a90>> being returned to pool
2024-03-02 12:49:13,475 DEBUG sqlalchemy.pool.impl.AsyncAdaptedQueuePool Connection <AdaptedConnection <asyncpg.connection.Connection object at 0x7f3d87c88a90>> rollback-on-return
```
