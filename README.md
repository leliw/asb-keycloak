# Angular - Spring Boot - Keycloak

This project is similar to https://github.com/leliw/asb-basic but authorization is based on Keycloak instead of
Spring Security.

## Development environment configuration

Install node from https://nodejs.org/ and then check instalation:
```bash
$ node --version
v16.13.2
```

Upgrade npm (globally) and install Angular CLI (locally)
```bash
$ npm install -g npm
$ npm install -g @angular/cli
```

Check ng command:
```bash
$ ng version
     _                      _                 ____ _     ___
    / \   _ __   __ _ _   _| | __ _ _ __     / ___| |   |_ _|
   / △ \ | '_ \ / _` | | | | |/ _` | '__|   | |   | |    | |
  / ___ \| | | | (_| | |_| | | (_| | |      | |___| |___ | |
 /_/   \_\_| |_|\__, |\__,_|_|\__,_|_|       \____|_____|___|
                |___/


Angular CLI: 13.2.0
Node: 16.13.2
Package Manager: npm 8.4.1
OS: win32 x64

Angular:
...

Package                      Version
------------------------------------------------------
@angular-devkit/architect    0.1302.0 (cli-only)
@angular-devkit/core         13.2.0 (cli-only)
@angular-devkit/schematics   13.2.0 (cli-only)
@schematics/angular          13.2.0 (cli-only)
```

I use also:
- Visual Studio Code: https://code.visualstudio.com/download 
- Eclipce IDE https://www.eclipse.org/downloads/

## Generate standard projects

Create parent folder for both projects (frontend and backend).

```bash
$ mkdir asb-keycloak
$ cd asb-keycloak
```

Create Angular project with two example components.
Answer questions:
- Would you like to add Angular routing? **Yes**
- Which stylesheet format would you like to use? **CSS**

```bash
$ ng new frontend
$ cd frontend
$ ng add @angular/material
$ ng generate @angular/material:navigation nav
$ ng generate @angular/material:dashboard home
```

You can run it.

```bash
ng serve --open
```

## Keycloak server

I've copied docker stack from https://github.com/mauriciovigolo/keycloak-angular/tree/eccbe1c70befe797193886e366dab9003d7e03eb/example. It's saved in `keycloak` directory.
```bash
cd ..
mkdir keycloak
cd keycloak
curl https://raw.githubusercontent.com/mauriciovigolo/keycloak-angular/eccbe1c70befe797193886e366dab9003d7e03eb/example/docker-compose.yml --output docker-compose.yml
curl https://raw.githubusercontent.com/mauriciovigolo/keycloak-angular/eccbe1c70befe797193886e366dab9003d7e03eb/example/Dockerfile --output Dockerfile
mkdir scripts
curl https://raw.githubusercontent.com/mauriciovigolo/keycloak-angular/eccbe1c70befe797193886e366dab9003d7e03eb/example/scripts/.gitattributes --output scripts/.gitattributes
curl https://raw.githubusercontent.com/mauriciovigolo/keycloak-angular/eccbe1c70befe797193886e366dab9003d7e03eb/example/scripts/create-client.sh --output scripts/create-client.sh
curl https://raw.githubusercontent.com/mauriciovigolo/keycloak-angular/eccbe1c70befe797193886e366dab9003d7e03eb/example/scripts/startup.sh --output scripts/startup.sh
```

Important is `scripts/create-client.sh` file. There is created client which is used in this tutorial. You can do the same manully of course.
```bash
#!/bin/bash
export PATH=$PATH:$JBOSS_HOME/bin

AUTH_ENDPOINT=http://localhost:8080/auth/
CLIENT_ID=asb-keycloak

while ! curl -s --head  --request GET $AUTH_ENDPOINT | grep "200 OK" > /dev/null; do
  echo "Waiting for Keycloak server..."
  sleep 5s
done

kcreg.sh config credentials --server $AUTH_ENDPOINT --realm master --user $KEYCLOAK_USER --password $KEYCLOAK_PASSWORD
client=$(kcreg.sh get $CLIENT_ID)

if [ -z "$client" ]; then
    kcreg.sh create -s clientId=$CLIENT_ID -s 'redirectUris=["http://localhost:4200/*", "http://localhost/*"]' -s 'publicClient=true'
else
    echo "The client has already been created."
fi
```
In the code above is important:
- AUTH_ENDPOINT=http://localhost:8080/auth/
- CLIENT_ID=asb-keycloak
- ...redirectUris=["http://localhost:4200/*", "http://localhost/*"]...

Start keycloak server hith command:
```bash
$ docker-compose up -d
$ cd ..
```

Now you can launch http://localhost:8080/auth/ and then go "Administration console" and login (u: admin, p: admin). In the "Clients" list you can see creaded "asb-keycloak" client.

## Keycloak client installation

```bash
$ cd ..
$ cd frontend
$ npm install keycloak-angular keycloak-js
$ code .
```

Add extra flag `allowSyntheticDefaultImports` to `tsconfig.json`.
```json
{
  "compileOnSave": false,
  "compilerOptions": {
    "allowSyntheticDefaultImports": true,
```

## Frontend implementation

Add Keycloak initializer provider in `app.module.ts`.
```typescript
function initializeKeycloak(keycloak: KeycloakService) {
  return () =>
    keycloak.init({
      config: {
        url: 'http://localhost:8080/auth',
        realm: 'master',
        clientId: 'asb-keycloak',
      },
      initOptions: {
        onLoad: 'check-sso',
        silentCheckSsoRedirectUri:
          window.location.origin + '/assets/silent-check-sso.html',
      },
    });
}
```

... and ...
```typescript
@NgModule({
  declarations: [ ... ],
  imports: [
    ...
    
    KeycloakAngularModule
  ],
  providers: [
    {
      provide: APP_INITIALIZER,
      useFactory: initializeKeycloak,
      multi: true,
      deps: [KeycloakService],
    },
  ],
```

Create a file called `silent-check-sso.html` in the `assets` directory of your application and paste in the contents as seen below.
```html
<html>
  <body>
    <script>
      parent.postMessage(location.href, location.origin);
    </script>
  </body>
</html>
```

Add KeycloakService and KeycloakProfile to `app.component.ts'.
```typescript
export class AppComponent implements OnInit {
  title = 'frontend';

  public isLoggedIn = false;
  public userProfile: KeycloakProfile | null = null;
  
  constructor(private readonly keycloak: KeycloakService, public router : Router) {
  }

  public async ngOnInit() {
    this.isLoggedIn = await this.keycloak.isLoggedIn();

    if (this.isLoggedIn) {
      this.userProfile = await this.keycloak.loadUserProfile();
    }
  }

  public login() {
    this.keycloak.login();
  }

  public logout() {
    this.keycloak.logout();
  }
}
```

Replace `app.component.html` with below code to show the difference.
```html
<button *ngIf="isLoggedIn" type="button" (click)="logout()">Log out</button>
<button *ngIf="!isLoggedIn" type="button" (click)="login()">Log in</button>
```

As you see keycloak login is work fine.
Now, we can merge keycloak with Angular Material Navgation component.

Reduce `app.component.html` to:
```html
<app-nav *ngIf="isLoggedIn"></app-nav>
```

Add auto login in `app.component.ts':
```typescript
  public async ngOnInit() {
    this.isLoggedIn = await this.keycloak.isLoggedIn();

    if (this.isLoggedIn) {
      this.userProfile = await this.keycloak.loadUserProfile();
    } else {
      this.login();
    }
  }
```

Add logout button in `nav.component.html`
```html
      <span>frontend</span>
      <span class="spacer"></span>
      <button mat-icon-button aria-label="Logout">
        <mat-icon aria-hidden="false" aria-label="Logout" (click)="logout()">logout</mat-icon>
      </button>      
    </mat-toolbar>
    <router-outlet></router-outlet>
  </mat-sidenav-content>
</mat-sidenav-container>
```
add logout method in `nav.component.ts`,
```typescript
  constructor(private breakpointObserver: BreakpointObserver, private app: AppComponent) {}

  logout() {
    this.app.logout();
  }
```
and finally add proper style in `nav.component.css`
```css
.spacer {
  flex: 1 1 auto;
}
```


## Backend - authorization

Create Spring Boot project with Spring Web and Spring Security dependencies. You can use https://start.spring.io/.

<img src="img/SpringInitializr-keycloak.png?raw=true" />

Save backend.zip in parent folder for both projects and unzip it.

```bash
cd ..
unzip backend.zip
cd backend

```

You can run it.

```bash
cd backend
mvn clean install
mvn spring-boot:run
```

Add in `pom.xml` keycloak dependency:
```xml
        <dependency>
            <groupId>org.keycloak</groupId>
            <artifactId>keycloak-spring-boot-starter</artifactId>
        </dependency>
	</dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.keycloak.bom</groupId>
                <artifactId>keycloak-adapter-bom</artifactId>
                <version>15.0.2</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

Set server port and keycloak properties in `application.properties`.
```properties
server.port=8090

keycloak.auth-server-url=http://localhost:8080/auth
keycloak.realm=master
keycloak.resource=asb-keycloak
keycloak.public-client=true
keycloak.principal-attribute=preferred_username
```

Create security configuration where all reqests should be authorized:
```java
@EnableWebSecurity
public class GlobalSecurityConfiguration extends KeycloakWebSecurityConfigurerAdapter  {

    @Bean
    @Override
    protected SessionAuthenticationStrategy sessionAuthenticationStrategy() {
        return new RegisterSessionAuthenticationStrategy(new SessionRegistryImpl());
    }

    @Override
    public void configure(AuthenticationManagerBuilder auth) {
        KeycloakAuthenticationProvider keycloakAuthenticationProvider = keycloakAuthenticationProvider();
        auth.authenticationProvider(keycloakAuthenticationProvider);
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        super.configure(http);
        http
                .authorizeRequests()
                .anyRequest()
                .authenticated();
    }

}
```

And simple controller showing logged user name:
```java
@RestController
public class HelloController {

    @GetMapping("/api/hello")
    public ResponseEntity<String> hello(Authentication authentication) {
        final String body = "Hello " + authentication.getName();
        return ResponseEntity.ok(body);
    }
}
```

And extra config resolving circular dependency:
```java
@Configuration
public class KeycloakConfiguration {

    @Bean
    public KeycloakSpringBootConfigResolver keycloakConfigResolver() {
        return new KeycloakSpringBootConfigResolver();
    }
}
```

Now you can run it by `mvn clean spring-boot:run` and check `http://localhost:8090/hello`.


## Developent environment (NGINX)

In production use there will be one server, but in development two separate servers are more convinient. 
As you can see before, there are two saparate servers working on ports 4200 (Angular) and 8090 (Spring Boot). 
In that case there is a problem with CORS (Cross-Origin Resource Sharing) and CSRF (Cross-site request forgery) 
protection. It is possible to configure Spring Boot Server to bypass these protections, but for me it is 
easier to use NGINX serwer in Docker container as a proxy.

Create `nginx-angular-dev8090` dictionary and two files inside.
`nginx.conf`:
```conf
events {
  worker_connections 768;
  # multi_accept on;
}

http {
	server {
		listen 80;

		server_name your.app;

		location /sso {
			proxy_pass       http://host.docker.internal:8090;
			proxy_set_header Upgrade    $http_upgrade;
			proxy_set_header Connection $http_connection;
			proxy_set_header Host       $host;
		}
		
		location /api {
			proxy_pass       http://host.docker.internal:8090;
			proxy_set_header Upgrade    $http_upgrade;
			proxy_set_header Connection $http_connection;
			proxy_set_header Host       $host;
		}
		
		location / {
			proxy_pass       http://host.docker.internal:4200;
			proxy_set_header Upgrade    $http_upgrade;
			proxy_set_header Connection $http_connection;
			proxy_set_header Host       $host;
		}
		
	}
}
```
and `Dockerfile`:
```Dockerfile
FROM nginx
COPY nginx.conf /etc/nginx/nginx.conf
```

Build image and run docker container:
```bash
cd nginx-angular-dev8090
docker build -t leliw/nginx-angular-dev8090 .
docker run -p 80:80 -d --name nginx-angular-dev8090 leliw/nginx-angular-dev8090
```

Now, at http://localhost/ is available Agnular application and at http://localhost/api/hello - Spring Boot.

## Angular - Spring Boot - Keycloak

Now let's show colaboration Angular - Spring Boot - Keycloak.

Add routing path in `app-routing.module.ts`
```typescript
const routes: Routes = [
  { path: '', redirectTo: '/home', pathMatch: 'full' },
  { path: 'home', component: HomeComponent }  
];
```

Modify `home.component.ts` to get data from server.
```typescript
  helloFromServer: string  = "";

  constructor(private breakpointObserver: BreakpointObserver, private http: HttpClient) {}

  public ngOnInit() {
    this.http.get("/api/hello", {responseType: 'text'}).subscribe(res => this.helloFromServer = res);
  }
```
and update card content in ``
```html
        <mat-card-content class="dashboard-card-content">
          <div>{{helloFromServer}}</div>
        </mat-card-content>
```

Now you can see server response in card contets. In Chrome you can also check authorization token send by
Angular frontend to server. Open developent tools and check request header for `hello` call.
There is `Authorization` header starts with `Bearer` keyword. Copy value of this heaed without `Bearer` 
and paste into https://jwt.io/. The token will be decoded and signature will be verified.


## All together in production environment

In production environment there is only one server. Angular resources are served by Java code as static files.
* Add config for static files
```java
@Configuration
public class BackendApplicationWebMvcConfigurer implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/**")
            .addResourceLocations("classpath:/static/")
            .resourceChain(true)
            .addResolver(new PathResourceResolver() {
                @Override
                protected Resource getResource(String resourcePath, Resource location) throws IOException {
                    Resource requestedResource = location.createRelative(resourcePath);
                    return requestedResource.exists() && requestedResource.isReadable() ? requestedResource : new ClassPathResource("/static/index.html");
                }
            });
    }
}
```
* Bilding Angular source by Maven. Add `pom.xml` file in `frontend` folder with frontend-maven-plugin plugin
```xml
            <plugin>
                <groupId>com.github.eirslett</groupId>
                <artifactId>frontend-maven-plugin</artifactId>
                <version>1.7.6</version>
                <configuration>
                    <workingDirectory>./</workingDirectory>
                    <nodeVersion>v16.13.2</nodeVersion>
                    <npmVersion>8.4.1</npmVersion>					
                </configuration>
                <executions>
                    <execution>
                        <id>install node and npm</id>
                        <goals>
                            <goal>install-node-and-npm</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>npm install</id>
                        <goals>
                            <goal>npm</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>npm run build</id>
                        <goals>
                            <goal>npm</goal>
                        </goals>
                        <configuration>
                            <arguments>run build</arguments>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
```

* Add copying Angular files do static floder in backend/pom.xml
```xml
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-resources-plugin</artifactId>
				<version>2.4.2</version>
				<executions>
					<execution>
						<id>default-copy-resources</id>
						<phase>process-resources</phase>
						<goals>
							<goal>copy-resources</goal>
						</goals>
						<configuration>
							<overwrite>false</overwrite>
							<outputDirectory>target/classes/static</outputDirectory>
							<resources>
								<resource>
									<directory>../frontend/dist/frontend</directory>
								</resource>
							</resources>
						</configuration>
					</execution>
				</executions>
			</plugin>
```

* Parent pom.xml file in main folder apoint both projects
```xml
  <packaging>pom</packaging>

  <modules>
    <module>frontend</module>
    <module>backend</module>
  </modules>
```

Then build and (stop docker first) run built jar.
```bash
mvn clean install
java -jar backend/target/backend-0.0.1-SNAPSHOT.jar
```
Application is available at http://localhost:8090/.
But ... it doesn't work :-(

Is because getting Angular code authorizing user needs already authrized user.
So change `GlobalSecurityConfiguration` to protect only `/api/` pages.
```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        super.configure(http);
        http
        .authorizeRequests()
            .antMatchers("/api").authenticated()
			.anyRequest().permitAll();
    }
```

That's all. Build and run again.


References:
1. https://www.npmjs.com/package/keycloak-angular
2. https://dzone.com/articles/secure-spring-boot-application-with-keycloak
3. https://jwt.io/
4. https://stackoverflow.com/questions/59289320/keycloak-check-sso-create
