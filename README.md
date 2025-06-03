Guia de Aplicação NestJS com Prisma + Docker: CRUD de Pokemon
Aqui está um guia passo a passo para desenvolver uma aplicação CRUD de Pokemon usando NestJS com Prisma, PostgreSQL e Docker, seguindo os princípios SOLID.

1. Configuração inicial do projeto
Primeiro, vamos instalar o NestJS CLI e criar um novo projeto:
bashnpm i -g @nestjs/cli

nest new pokemon-api
# Ou se já estiver dentro da pasta
nest new .

cd pokemon-api

code .
Instale as dependências necessárias:
bashnpm install @nestjs/swagger swagger-ui-express @prisma/client prisma
npm install -D prisma @types/node

# Dependências para validação
npm install class-validator class-transformer

2. Configuração do Docker
Crie o arquivo Dockerfile:
dockerfileFROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

RUN npx prisma generate
RUN npm run build

EXPOSE 3000

CMD ["npm", "run", "start:prod"]
Crie o arquivo docker-compose.yml:
yamlversion: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://pokemon_user:pokemon_pass@db:5432/pokemon_db
    depends_on:
      - db
    volumes:
      - .:/app
      - /app/node_modules

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: pokemon_user
      POSTGRES_PASSWORD: pokemon_pass
      POSTGRES_DB: pokemon_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
Crie o arquivo .env:
envDATABASE_URL="postgresql://pokemon_user:pokemon_pass@localhost:5432/pokemon_db"

3. Configuração do Prisma
Inicialize o Prisma:
bashnpx prisma init
Edite o arquivo prisma/schema.prisma:
prismagenerator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Pokemon {
  id        String   @id @default(cuid())
  name      String   @unique
  type      String
  level     Int
  hp        Int
  attack    Int
  defense   Int
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@map("pokemons")
}
Execute as migrations:
bashnpx prisma migrate dev --name init
npx prisma generate

4. Configuração do Swagger
Configure o Swagger no arquivo src/main.ts:
typescriptimport { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
  }));

  const config = new DocumentBuilder()
    .setTitle('Pokemon API')
    .setDescription('API para gerenciamento de Pokémons')
    .setVersion('1.0')
    .addTag('pokemon')
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document);

  await app.listen(3000);
  console.log('Application is running on: http://localhost:3000');
  console.log('Swagger documentation: http://localhost:3000/api');
}
bootstrap();

5. Criação dos DTOs
Crie a estrutura de diretórios:
bashnest g module pokemon
nest g controller pokemon
nest g service pokemon
Crie o arquivo src/pokemon/dto/create-pokemon.dto.ts:
typescriptimport { ApiProperty } from '@nestjs/swagger';
import { IsString, IsInt, Min, Max, IsNotEmpty } from 'class-validator';

export class CreatePokemonDto {
  @ApiProperty({ example: 'Pikachu', description: 'Nome do Pokemon' })
  @IsString()
  @IsNotEmpty()
  name: string;

  @ApiProperty({ example: 'Electric', description: 'Tipo do Pokemon' })
  @IsString()
  @IsNotEmpty()
  type: string;

  @ApiProperty({ example: 25, description: 'Nível do Pokemon', minimum: 1, maximum: 100 })
  @IsInt()
  @Min(1)
  @Max(100)
  level: number;

  @ApiProperty({ example: 35, description: 'HP do Pokemon', minimum: 1 })
  @IsInt()
  @Min(1)
  hp: number;

  @ApiProperty({ example: 55, description: 'Ataque do Pokemon', minimum: 1 })
  @IsInt()
  @Min(1)
  attack: number;

  @ApiProperty({ example: 40, description: 'Defesa do Pokemon', minimum: 1 })
  @IsInt()
  @Min(1)
  defense: number;
}
Crie o arquivo src/pokemon/dto/update-pokemon.dto.ts:
typescriptimport { PartialType } from '@nestjs/swagger';
import { CreatePokemonDto } from './create-pokemon.dto';

export class UpdatePokemonDto extends PartialType(CreatePokemonDto) {}
Crie o arquivo src/pokemon/entities/pokemon.entity.ts:
typescriptimport { ApiProperty } from '@nestjs/swagger';

export class Pokemon {
  @ApiProperty({ example: 'cuid1234567890', description: 'ID único do Pokemon' })
  id: string;

  @ApiProperty({ example: 'Pikachu', description: 'Nome do Pokemon' })
  name: string;

  @ApiProperty({ example: 'Electric', description: 'Tipo do Pokemon' })
  type: string;

  @ApiProperty({ example: 25, description: 'Nível do Pokemon' })
  level: number;

  @ApiProperty({ example: 35, description: 'HP do Pokemon' })
  hp: number;

  @ApiProperty({ example: 55, description: 'Ataque do Pokemon' })
  attack: number;

  @ApiProperty({ example: 40, description: 'Defesa do Pokemon' })
  defense: number;

  @ApiProperty({ description: 'Data de criação' })
  createdAt: Date;

  @ApiProperty({ description: 'Data de atualização' })
  updatedAt: Date;
}

6. Configuração do Prisma Service
Crie o arquivo src/prisma/prisma.service.ts:
typescriptimport { Injectable, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
Crie o arquivo src/prisma/prisma.module.ts:
typescriptimport { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}

7. Repository Pattern (SOLID)
Crie a interface do repositório src/pokemon/repository/pokemon-repository.interface.ts:
typescriptimport { CreatePokemonDto } from '../dto/create-pokemon.dto';
import { UpdatePokemonDto } from '../dto/update-pokemon.dto';
import { Pokemon } from '../entities/pokemon.entity';

export abstract class PokemonRepository {
  abstract findAll(): Promise<Pokemon[]>;
  abstract findById(id: string): Promise<Pokemon | null>;
  abstract findByName(name: string): Promise<Pokemon | null>;
  abstract create(createPokemonDto: CreatePokemonDto): Promise<Pokemon>;
  abstract update(id: string, updatePokemonDto: UpdatePokemonDto): Promise<Pokemon | null>;
  abstract remove(id: string): Promise<Pokemon | null>;
}
Crie a implementação src/pokemon/repository/pokemon-repository.prisma.ts:
typescriptimport { Injectable } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service';
import { PokemonRepository } from './pokemon-repository.interface';
import { CreatePokemonDto } from '../dto/create-pokemon.dto';
import { UpdatePokemonDto } from '../dto/update-pokemon.dto';
import { Pokemon } from '../entities/pokemon.entity';

@Injectable()
export class PokemonRepositoryPrisma implements PokemonRepository {
  constructor(private readonly prisma: PrismaService) {}

  async findAll(): Promise<Pokemon[]> {
    return this.prisma.pokemon.findMany({
      orderBy: { createdAt: 'desc' },
    });
  }

  async findById(id: string): Promise<Pokemon | null> {
    return this.prisma.pokemon.findUnique({
      where: { id },
    });
  }

  async findByName(name: string): Promise<Pokemon | null> {
    return this.prisma.pokemon.findUnique({
      where: { name },
    });
  }

  async create(createPokemonDto: CreatePokemonDto): Promise<Pokemon> {
    return this.prisma.pokemon.create({
      data: createPokemonDto,
    });
  }

  async update(id: string, updatePokemonDto: UpdatePokemonDto): Promise<Pokemon | null> {
    try {
      return await this.prisma.pokemon.update({
        where: { id },
        data: updatePokemonDto,
      });
    } catch (error) {
      return null;
    }
  }

  async remove(id: string): Promise<Pokemon | null> {
    try {
      return await this.prisma.pokemon.delete({
        where: { id },
      });
    } catch (error) {
      return null;
    }
  }
}

8. Service
Edite o arquivo src/pokemon/pokemon.service.ts:
typescriptimport { Injectable, NotFoundException, ConflictException } from '@nestjs/common';
import { PokemonRepository } from './repository/pokemon-repository.interface';
import { CreatePokemonDto } from './dto/create-pokemon.dto';
import { UpdatePokemonDto } from './dto/update-pokemon.dto';
import { Pokemon } from './entities/pokemon.entity';

@Injectable()
export class PokemonService {
  constructor(private readonly pokemonRepository: PokemonRepository) {}

  async create(createPokemonDto: CreatePokemonDto): Promise<Pokemon> {
    const existingPokemon = await this.pokemonRepository.findByName(createPokemonDto.name);
    
    if (existingPokemon) {
      throw new ConflictException(`Pokemon com nome '${createPokemonDto.name}' já existe`);
    }

    return this.pokemonRepository.create(createPokemonDto);
  }

  async findAll(): Promise<Pokemon[]> {
    return this.pokemonRepository.findAll();
  }

  async findOne(id: string): Promise<Pokemon> {
    const pokemon = await this.pokemonRepository.findById(id);
    
    if (!pokemon) {
      throw new NotFoundException(`Pokemon com ID '${id}' não encontrado`);
    }

    return pokemon;
  }

  async update(id: string, updatePokemonDto: UpdatePokemonDto): Promise<Pokemon> {
    if (updatePokemonDto.name) {
      const existingPokemon = await this.pokemonRepository.findByName(updatePokemonDto.name);
      if (existingPokemon && existingPokemon.id !== id) {
        throw new ConflictException(`Pokemon com nome '${updatePokemonDto.name}' já existe`);
      }
    }

    const pokemon = await this.pokemonRepository.update(id, updatePokemonDto);
    
    if (!pokemon) {
      throw new NotFoundException(`Pokemon com ID '${id}' não encontrado`);
    }

    return pokemon;
  }

  async remove(id: string): Promise<Pokemon> {
    const pokemon = await this.pokemonRepository.remove(id);
    
    if (!pokemon) {
      throw new NotFoundException(`Pokemon com ID '${id}' não encontrado`);
    }

    return pokemon;
  }
}

9. Controller
Edite o arquivo src/pokemon/pokemon.controller.ts:
typescriptimport { Controller, Get, Post, Body, Patch, Param, Delete, HttpCode, HttpStatus } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse, ApiParam } from '@nestjs/swagger';
import { PokemonService } from './pokemon.service';
import { CreatePokemonDto } from './dto/create-pokemon.dto';
import { UpdatePokemonDto } from './dto/update-pokemon.dto';
import { Pokemon } from './entities/pokemon.entity';

@ApiTags('pokemon')
@Controller('pokemon')
export class PokemonController {
  constructor(private readonly pokemonService: PokemonService) {}

  @Post()
  @ApiOperation({ summary: 'Criar um novo Pokemon' })
  @ApiResponse({ status: 201, description: 'Pokemon criado com sucesso.', type: Pokemon })
  @ApiResponse({ status: 409, description: 'Pokemon com este nome já existe.' })
  create(@Body() createPokemonDto: CreatePokemonDto): Promise<Pokemon> {
    return this.pokemonService.create(createPokemonDto);
  }

  @Get()
  @ApiOperation({ summary: 'Listar todos os Pokemon' })
  @ApiResponse({ status: 200, description: 'Lista de Pokemon retornada com sucesso.', type: [Pokemon] })
  findAll(): Promise<Pokemon[]> {
    return this.pokemonService.findAll();
  }

  @Get(':id')
  @ApiOperation({ summary: 'Buscar Pokemon por ID' })
  @ApiParam({ name: 'id', description: 'ID do Pokemon' })
  @ApiResponse({ status: 200, description: 'Pokemon encontrado.', type: Pokemon })
  @ApiResponse({ status: 404, description: 'Pokemon não encontrado.' })
  findOne(@Param('id') id: string): Promise<Pokemon> {
    return this.pokemonService.findOne(id);
  }

  @Patch(':id')
  @ApiOperation({ summary: 'Atualizar Pokemon' })
  @ApiParam({ name: 'id', description: 'ID do Pokemon' })
  @ApiResponse({ status: 200, description: 'Pokemon atualizado com sucesso.', type: Pokemon })
  @ApiResponse({ status: 404, description: 'Pokemon não encontrado.' })
  @ApiResponse({ status: 409, description: 'Pokemon com este nome já existe.' })
  update(@Param('id') id: string, @Body() updatePokemonDto: UpdatePokemonDto): Promise<Pokemon> {
    return this.pokemonService.update(id, updatePokemonDto);
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  @ApiOperation({ summary: 'Remover Pokemon' })
  @ApiParam({ name: 'id', description: 'ID do Pokemon' })
  @ApiResponse({ status: 204, description: 'Pokemon removido com sucesso.' })
  @ApiResponse({ status: 404, description: 'Pokemon não encontrado.' })
  async remove(@Param('id') id: string): Promise<void> {
    await this.pokemonService.remove(id);
  }
}

10. Configuração dos Módulos
Edite o arquivo src/pokemon/pokemon.module.ts:
typescriptimport { Module } from '@nestjs/common';
import { PokemonService } from './pokemon.service';
import { PokemonController } from './pokemon.controller';
import { PokemonRepository } from './repository/pokemon-repository.interface';
import { PokemonRepositoryPrisma } from './repository/pokemon-repository.prisma';

@Module({
  controllers: [PokemonController],
  providers: [
    PokemonService,
    {
      provide: PokemonRepository,
      useClass: PokemonRepositoryPrisma,
    },
  ],
})
export class PokemonModule {}
Edite o arquivo src/app.module.ts:
typescriptimport { Module } from '@nestjs/common';
import { PrismaModule } from './prisma/prisma.module';
import { PokemonModule } from './pokemon/pokemon.module';

@Module({
  imports: [PrismaModule, PokemonModule],
})
export class AppModule {}

11. Executando a aplicação
Para desenvolvimento local:
bash# Subir apenas o banco de dados
docker-compose up db -d

# Executar migrations
npx prisma migrate dev

# Iniciar a aplicação
npm run start:dev
Para executar com Docker completo:
bash# Construir e executar todos os serviços
docker-compose up --build

# Em outro terminal, executar as migrations
docker-compose exec app npx prisma migrate deploy

12. Testando a API
Acesse http://localhost:3000/api para ver a documentação Swagger.
Exemplo de requisições:
Criar Pokemon:
jsonPOST /pokemon
{
  "name": "Pikachu",
  "type": "Electric",
  "level": 25,
  "hp": 35,
  "attack": 55,
  "defense": 40
}
Listar todos:
GET /pokemon
Buscar por ID:
GET /pokemon/{id}
Atualizar:
jsonPATCH /pokemon/{id}
{
  "level": 30,
  "hp": 40
}
Remover:
DELETE /pokemon/{id}

13. Comandos úteis
bash# Resetar banco de dados
npx prisma migrate reset

# Visualizar banco de dados
npx prisma studio

# Gerar cliente Prisma após mudanças no schema
npx prisma generate

# Criar nova migration
npx prisma migrate dev --name nome_da_migration

# Deploy das migrations em produção
npx prisma migrate deploy

Conclusão
Esta aplicação segue os princípios SOLID com separação clara de responsabilidades:

Single Responsibility: Cada classe tem uma única responsabilidade
Open/Closed: Uso de interfaces permite extensão sem modificação
Liskov Substitution: Repository pode ser substituído por diferentes implementações
Interface Segregation: Interfaces específicas para cada necessidade
Dependency Inversion: Service depende de abstração, não de implementação concreta

A arquitetura em camadas (Controller → Service → Repository) garante baixo acoplamento e alta coesão, facilitando manutenção e testes.
