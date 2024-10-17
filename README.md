# Tradutor-de-Artigos-T-cnicos-com-AzureAI
Tradução de texto usando o Azure AI Translator em um aplicativo NestJS
Introdução
Sou um estudante de idiomas que aprende mandarim e espanhol no meu tempo livre. Quando descobri que a tradução de texto usando o Azure AI é possível, quis aproveitar a força do serviço de tradução do Azure Cognitive Service no meu hobby. Portanto, criei um aplicativo NestJS para traduzir textos entre dois idiomas por meio do serviço de tradução do Azure AI.

Inscreva-se gratuitamente no Azure AI Service
Acesse https://azure.microsoft.com/en-us/free/ai-services/ para criar uma conta gratuita.
Crie um serviço de tradutor no Portal do Azure
Clique em APIs e chaves no menu e copie a chave da API, o local e o URL de tradução
Crie um novo projeto NestJS
nest new nestjs-genai-translation
Instalar dependências
npm i --save-exact  zod @nestjs/swagger @nestjs/throttler dotenv compression helmet
Gerar um módulo de tradução
nest g mo translation
nest g co translation/presenters/http/translator --flat
nest g s translation/application/azureTranslator --flat
Crie um módulo de tradução, um controlador e um serviço para a API.

Definir variáveis ​​de ambiente do Azure OpenAI
// .env.example

PORT=3000
AZURE_OPENAI_TRANSLATOR_API_KEY=<translator api key>
AZURE_OPENAI_TRANSLATOR_URL=<translator url>/translate
AZURE_OPENAI_TRANSLATOR_API_VERSION="3.0"
AZURE_OPENAI_LOCATION=eastasia
AI_SERVICE=azureOpenAI
Copie .env.examplepara .enve substitua AZURE_OPENAI_TRANSLATOR_API_KEYe AZURE_OPENAI_TRANSLATOR_URLpela chave de API real e pelo URL do tradutor, respectivamente.

AZURE_OPENAI_TRANSLATOR_API_KEY - Chave de API do serviço de tradução do Azure OpenAI
AZURE_OPENAI_TRANSLATOR_URL - URL do tradutor do serviço de tradução do Azure OpenAI
AZURE_OPENAI_TRANSLATOR_API_VERSION - Versão da API do serviço de tradução do Azure OpenAI
AZURE_OPENAI_LOCATION - Localização do serviço de tradução do Azure OpenAI
AI_SERVICE - Serviço de IA generativo a ser usado no aplicativo
Adicione .envao .gitignorearquivo para garantir que não enviemos acidentalmente a chave da API do Azure OpenAI Translator para o repositório do GitHub.

Adicionar arquivos de configuração
O projeto tem três arquivos de configuração. validate.config.tsvalida se a carga útil é válida antes que qualquer solicitação possa ser roteada para o controlador para execução

// validate.config.ts

import { ValidationPipe } from '@nestjs/common';

export const validateConfig = new ValidationPipe({
  whitelist: true,
  stopAtFirstError: true,
  forbidUnknownValues: false,
});
env.config.tsextrai as variáveis ​​de ambiente process.enve armazena os valores no objeto env.

// env.config.ts

import dotenv from 'dotenv';
import { Integration } from '~core/types/integration.type';

dotenv.config();

export const env = {
  PORT: parseInt(process.env.PORT || '3000'),
  AZURE_OPENAI_TRANSLATOR: {
    KEY: process.env.AZURE_OPENAI_TRANSLATOR_API_KEY || '',
    URL: process.env.AZURE_OPENAI_TRANSLATOR_URL || '',
    LOCATION: process.env.AZURE_OPENAI_LOCATION || 'eastasia',
    VERSION: process.env.AZURE_OPENAI_TRANSLATOR_API_VERSION || '3.0',
  },
  AI_SERVICE: (process.env.AI_SERVICE || 'azureOpenAI') as Integration,
};
throttler.config.tsdefine o limite de taxa da API de tradução.

// throttler.config.ts

import { ThrottlerModule } from '@nestjs/throttler';

export const throttlerConfig = ThrottlerModule.forRoot([
  {
    ttl: 60000,
    limit: 10,
  },
]);
Cada rota permite dez solicitações em 60.000 milissegundos ou 1 minuto.

Inicialize o aplicativo
// bootstrap.ts

export class Bootstrap {
  private app: NestExpressApplication;

  async initApp() {
    this.app = await NestFactory.create(AppModule);
  }

  enableCors() {
    this.app.enableCors();
  }

  setupMiddleware() {
    this.app.use(express.json({ limit: '1000kb' }));
    this.app.use(express.urlencoded({ extended: false }));
    this.app.use(compression());
    this.app.use(helmet());
  }

  setupGlobalPipe() {
    this.app.useGlobalPipes(validateConfig);
  }

  async startApp() {
    await this.app.listen(env.PORT);
  }

  setupSwagger() {
    const config = new DocumentBuilder()
      .setTitle('Generative AI Translator')
      .setDescription('Integrate with Generative AI to translate a text from one language to another language')
      .setVersion('1.0')
      .addTag('Azure OpenAI, Langchain Gemini AI Model, Google Translate Cloud API')
      .build();
    const document = SwaggerModule.createDocument(this.app, config);
    SwaggerModule.setup('api', this.app, document);
  }
}
Adicione uma Bootstrapclasse para configurar o Swagger, middleware, validação global, cors e, finalmente, inicialização do aplicativo.

// main.ts

import { Bootstrap } from '~core/bootstrap';

async function bootstrap() {
  const bootstrap = new Bootstrap();
  await bootstrap.initApp();
  bootstrap.enableCors();
  bootstrap.setupMiddleware();
  bootstrap.setupGlobalPipe();
  bootstrap.setupSwagger();
  await bootstrap.startApp();
}
bootstrap()
  .then(() => console.log('The application starts successfully'))
  .catch((error) => console.error(error));
A bootstrapfunção habilita o CORS, registra o middleware no aplicativo, configura a documentação do Swagger e usa um pipe global para validar cargas úteis.

Estabeleci as bases e o próximo passo é adicionar rotas para receber carga útil e traduzir textos entre o idioma de origem e o idioma de destino.

Definir Tradução DTO
// languages.enum

import { z } from 'zod';

const LANGUAGE_CODES = {
  English: 'en',
  Spanish: 'es',
  'Simplified Chinese': 'zh-Hans',
  'Traditional Chinese': 'zh-Hant',
  Vietnamese: 'vi',
  Japanese: 'ja',
} as const;

export const ZOD_LANGUAGE_CODES = z.nativeEnum(LANGUAGE_CODES, {
  required_error: 'Language code is required',
  invalid_type_error: 'Language code is invalid',
});
export type LanguageCodeType = z.infer<typeof ZOD_LANGUAGE_CODES>;
// translate-text.dto.ts

import { z } from 'zod';
import { ZOD_LANGUAGE_CODES } from '~translation/application/enums/languages.enum';

export const translateTextSchema = z
  .object({
    text: z.string({
      required_error: 'Text is required',
    }),
    srcLanguageCode: ZOD_LANGUAGE_CODES,
    targetLanguageCode: ZOD_LANGUAGE_CODES,
  })
  .required();

export type TranslateTextDto = z.infer<typeof translateTextSchema>;
translateTextSchemaaceita um texto, um código de idioma de origem e um código de idioma de destino. Então, eu uso zodpara inferir o tipo de translateTextSchema e atribuí-lo a TranslateTextDto.

Implementar o serviço Azure Translator
Este aplicativo também suportará langchain.js, Google Gemini e Google Translate Cloud API para traduzir textos. Portanto, criei uma interface Translator, e todos os serviços que implementam a interface devem cumprir o contrato.

//  translator-input.interface.ts

import { LanguageCodeType } from '../enums/languages.enum';

export interface TranslateInput {
  text: string;
  srcLanguageCode: LanguageCodeType;
  targetLanguageCode: LanguageCodeType;
}
// translate-result.interface.ts

import { Integration } from '~core/types/integration.type';

export interface TranslationResult {
  text: string;
  aiService: Integration;
}
// translator.interface.ts

export interface Translator {
  translate(input: TranslateInput): Promise<TranslationResult>;
}
// azure-translator.service.ts

type AzureTranslateResponse = {
  translations: [
    {
      text: string;
      to: string;
    },
  ];
};

@Injectable()
export class AzureTranslatorService implements Translator {
  constructor(private httpService: HttpService) {}

  async translate({ text, srcLanguageCode, targetLanguageCode }: TranslateInput): Promise<TranslationResult> {
    const data = [{ text }];
    const result$ = this.httpService
      .post<AzureTranslateResponse[]>(env.AZURE_OPENAI_TRANSLATOR.URL, data, {
        headers: {
          'Ocp-Apim-Subscription-Key': env.AZURE_OPENAI_TRANSLATOR.KEY,
          'Ocp-Apim-Subscription-Region': env.AZURE_OPENAI_TRANSLATOR.LOCATION,
          'Content-type': 'application/json',
          'X-ClientTraceId': v4(),
        },
        params: {
           'api-version': env.AZURE_OPENAI_TRANSLATOR.VERSION,
           from: srcLanguageCode,
           to: targetLanguageCode,
        },
        responseType: 'json',
      })
      .pipe(
         map(({ data }) => data?.[0]?.translations?.[0].text || 'No result'),
         map((text) => ({
           text,
           aiService: <Integration>'azureOpenAI',
         })),
      );
    return firstValueFrom(result$);
  }
}
O método translate do AzureTranslatorService faz uma solicitação POST para traduzir o texto do código do idioma de origem para o código do idioma de destino. O httpServiceretorna um Observable; portanto, o Observable é passado para o firstValueFromoperador RxJS para converter em uma Promise.

Implementar o Translator Controller
// zod-validation.pipe.ts

export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown) {
    try {
      const parsedValue = this.schema.parse(value);
      return parsedValue;
    } catch (error) {
      console.error(error);
      if (error instanceof ZodError) {
        throw new BadRequestException(error.errors?.[0]?.message || 'Validation failed');
      } else if (error instanceof Error) {
        throw new BadRequestException(error.message);
      }
      throw error;
    }
  }
}
ZodValidationPipeé um pipe que valida o payload contra o esquema Zod. Quando a validação for bem-sucedida, o payload será analisado e retornado. Quando a validação falhar, o pipe intercepta o ZodErrore retorna uma instância de BadRequestException.

// translator.controller.ts

// omit the import statements to save space

@ApiTags('Translator')
@Controller('translator')
export class TranslatorController {
  constructor(@Inject(TRANSLATOR) private translatorService: Translator) {}

  @ApiBody({
    description: 'An intance of TranslatTextDto',
    required: true,
    schema: {
      type: 'object',
      properties: {
        text: {
          type: 'string',
          description: 'text to be translated',
        },
        srcLanguageCode: {
          type: 'string',
          description: 'source language code',
          enum: ['en', 'es', 'zh-Hans', 'zh-Hant', 'vi', 'ja'],
        },
        targetLanguageCode: {
          type: 'string',
          description: 'target language code',
          enum: ['en', 'es', 'zh-Hans', 'zh-Hant', 'vi', 'ja'],
        },
      },
    },
    examples: {
      greeting: {
        value: {
          text: 'Good morning, good afternoon, good evening.',
          srcLanguageCode: 'en',
          targetLanguageCode: 'es',
        },
      },
    },
  })
  @ApiResponse({
    description: 'The translated text',
    schema: {
      type: 'object',
      properties: {
        text: { type: 'string', description: 'translated text' },
        aiService: { type: 'string', description: 'AI service' },
      },
    },
    status: 200,
  })
  @HttpCode(200)
  @Post()
  @UsePipes(new ZodValidationPipe(translateTextSchema))
  translate(@Body() dto: TranslateTextDto): Promise<TranslationResult> {
    return this.translatorService.translate(dto);
  }
}
O TranslatorControllerinjects Translator, que é uma instância de AzureTranslatorService. O ponto de extremidade invoca o translatemétodo para executar a tradução de texto usando o Azure OpenAI.

Registro dinâmico
Este aplicativo registra o serviço de tradução com base na AI_SERVICEvariável de ambiente. O valor da variável de ambiente é um de azureOpenAI, langchain_googleChatModel e google_translate.

// .env.example

AI_SERVICE=azureOpenAI
// integration.type.ts

export type Integration = 'azureOpenAI' | 'langchain_googleChatModel' | 'google_translate';
// translator.module.ts

@Module({
  imports: [HttpModule],
  controllers: [TranslatorController],
})
export class TranslationModule {
  static register(type: Integration = 'azureOpenAI'): DynamicModule {
    const serviceMap = new Map<Integration, any>();
    serviceMap.set('azureOpenAI', AzureTranslatorService);
    const translatorService = serviceMap.get(type) || AzureTranslatorService;

    const providers: Provider[] = [
      {
        provide: TRANSLATOR,
        useClass: translatorService,
      },
    ];

    return {
      module: TranslationModule,
      providers,
    };
  }
}
Em TranslationModule, defino um método de registro que retorna um DynamicModule. Quando typeé azureOpenAI, o TRANSLATORtoken fornece AzureTranslatorService. Em seguida, TranslationModule.register(env.AI_SERVICE)cria um TranslationModuleque eu importo no AppModule.

// app.module.ts

@Module({
  imports: [throttlerConfig, TranslationModule.register(env.AI_SERVICE)],
  controllers: [AppController],
  providers: [
    AppService,
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard,
    },
  ],
})
export class AppModule {}
Teste os endpoints
Depois de iniciar o aplicativo, posso testar os endpoints com a documentação do cURL, Postman ou Swagger.

npm run start:dev
O URL da documentação do Swagger é http://localhost:3000/api .

Dockerize o aplicativo
// .dockerignore

.git
.gitignore
node_modules/
dist/
Dockerfile
.dockerignore
npm-debug.log
Crie um .dockerignorearquivo para o Docker ignorar alguns arquivos e diretórios.

// Dockerfile

# Use an official Node.js runtime as the base image
FROM node:20-alpine

# Set the working directory in the container
WORKDIR /app

# Copy package.json and package-lock.json to the working directory
COPY package*.json ./

# Install the dependencies
RUN npm install

RUN npm run build

# Copy the rest of the application code to the working directory
COPY . .

# Expose a port (if your application listens on a specific port)
EXPOSE 3000

# Define the command to run your application
CMD [ "npm", "start" ]
Adicionei o Dockerfileque instala as dependências, compila o aplicativo NestJS e o inicia.

//  .env.docker.example

PORT=3000
AZURE_OPENAI_TRANSLATOR_API_KEY=<translator api key>
AZURE_OPENAI_TRANSLATOR_URL=<translator url>/translate
AZURE_OPENAI_TRANSLATOR_API_VERSION="3.0"
AZURE_OPENAI_LOCATION=eastasia
GOOGLE_GEMINI_API_KEY=<google gemini api key>
GOOGLE_GEMINI_MODEL=gemini-pro
AI_SERVICE=langchain_googleChatModel
.env.docker.examplearmazena as variáveis ​​de ambiente relevantes que copiei do aplicativo NestJS.

// docker-compose.yaml

version: '3.8'

services:
  backend:
    build:
      context: ./nestjs-genai-translation
      dockerfile: Dockerfile
    environment:
      - PORT=${PORT}
      - AZURE_OPENAI_TRANSLATOR_API_KEY=${AZURE_OPENAI_TRANSLATOR_API_KEY}
      - AZURE_OPENAI_TRANSLATOR_URL=${AZURE_OPENAI_TRANSLATOR_URL}
      - AZURE_OPENAI_TRANSLATOR_API_VERSION=${AZURE_OPENAI_TRANSLATOR_API_VERSION}
      - AZURE_OPENAI_LOCATION=${AZURE_OPENAI_LOCATION}
      - GOOGLE_GEMINI_API_KEY=${GOOGLE_GEMINI_API_KEY}
      - GOOGLE_GEMINI_MODEL=${GOOGLE_GEMINI_MODEL}
    ports:
      - "${PORT}:${PORT}"
    networks:
      - ai
    restart: always

networks:
  ai:
Adicionei o docker-compose.yamlna pasta raiz, que foi responsável por criar o contêiner do aplicativo NestJS.

Isso conclui meu post de blog sobre o uso do Azure OpenAI Translator Service para resolver um problema do mundo real. Eu apenas arranhei a superfície do Azure OpenAI, e a Microsoft oferece muitos serviços. Espero que você goste do conteúdo e continue acompanhando minha experiência de aprendizado em Angular, NestJS e outras tecnologias.

Recursos:
Repositório Github: https://github.com/railsstudent/fullstack-genai-translation/tree/main/nestjs-genai-translation
Serviço de tradutor do Azure OpenAI: https://learn.microsoft.com/en-us/azure/ai-services/translator/text-translation-overview
