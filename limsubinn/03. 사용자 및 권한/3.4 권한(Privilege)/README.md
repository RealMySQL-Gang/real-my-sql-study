## 3.4 권한(Privilege)

✅ MySQL은 Global 권한과 객체 단위의 권한으로 구분된다.

- 글로벌 권한: 데이터베이스나 테이블 이외의 객체에 적용되는 권한
  - `GRANT` 명령에서 특정 객체를 명시하지 말아야 한다.
  - 글로벌로 `ALL`이 사용되면 글로벌 수준에서 가능한 모든 권한을 부여한다.

- 객체 권한: 데이터베이스나 테이블을 제어하는 데 필요한 권한
  - `GRANT` 명령으로 권한을 부여할 때 반드시 특정 객체를 명시해야 한다.
  - 특정 객체에 `ALL` 권한이 부여되면 해당 객체에 적용될 수 있는 모든 객체 권한을 부여한다.

<br>

✅ 정적 권한과 동적 권한

- 정적 권한: MySQL 서버의 소스코드에 고정적으로 명시되어 있는 권한

- 동적 권한: MySQL 서버의 컴포넌트나 플러그인이 설치되면 그때 등록되는 권한

<br>

✅ 사용자에게 권한을 부여할 때는 `GRANT` 명령을 사용한다.

```sql
mysql> GRANT privilege_list ON db.table TO 'user'@'host';
```

- `GRANT OPTION` 권한은 GRANT 명령의 마지막에 `WITH GRANT OPTION`을 명시해서 부여한다.

- `privilege_list`에는 구분자(,)를 써서 권한 여러 개를 동시에 명시할 수 있다.

- `TO` 키워드 뒤에는 권한을 부여할 대상 사용자를 명시하고, `ON` 키워드 뒤에는 어떤 DB의 어떤 오브젝트에 권한을 부여할지 결정할 수 있다.
  - **글로벌 권한**은 GRANT 명령의 ON절에 항상 `*.*`를 사용한다. (특정 DB나 테이블에 권한을 부여할 수 없기 때문)
  - **DB 권한**은 GRANT 명령의 ON절에 `*.*`와 `db.*`를 사용할 수 있다. (특정 DB 또는 서버에 존재하는 모든 DB에 대해 권한을 부여할 수 있기 때문)
  - **테이블 권한**은 GRANT 명령의 ON절에 `*.*`와 `db.*`, `db.table`를 사용할 수 있다.
