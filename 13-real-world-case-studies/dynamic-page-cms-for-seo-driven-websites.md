
# Designing a Dynamic Page CMS for SEO-Driven Websites

> A real-world backend and system design case study about building a flexible Page CMS that allows admins to create, manage, publish, and optimize website pages without hardcoding every route.

---

## Table of Contents

* [Introduction](#introduction)
* [The Problem](#the-problem)
* [The Goal](#the-goal)
* [System Overview](#system-overview)
* [Page CMS Core Features](#page-cms-core-features)
* [Folder Structure](#folder-structure)
* [Page Domain Model](#page-domain-model)
* [Page Section Model](#page-section-model)
* [DTOs](#dtos)
* [Repository Contract](#repository-contract)
* [Slug Service](#slug-service)
* [Use Cases](#use-cases)
* [Application Service](#application-service)
* [MongoDB Schema](#mongodb-schema)
* [Mongo Repository Implementation](#mongo-repository-implementation)
* [Express Controller](#express-controller)
* [Routes](#routes)
* [Frontend Dynamic Rendering](#frontend-dynamic-rendering)
* [API Flow](#api-flow)
* [Problems Solved](#problems-solved)
* [Important Implementation Notes](#important-implementation-notes)
* [Conclusion](#conclusion)

---

## Introduction

In many real-world web applications, content pages are created directly inside the frontend codebase.

For example:

```txt
/app/about/page.tsx
/app/services/page.tsx
/app/used-iron-beam/page.tsx
/app/landing/summer-campaign/page.tsx
```

This approach works at the beginning.

But as the business grows, the number of pages increases.

Soon, every new SEO page, landing page, campaign page, service page, or product category page requires:

* a developer,
* a new route,
* new components,
* content updates,
* testing,
* deployment.

This creates unnecessary dependency between the content team and the engineering team.

To solve this, I designed a **Dynamic Page CMS**.

The goal was to move page creation from **hardcoded frontend files** to a **structured backend-driven content model**.

---

## The Problem

Before implementing a Page CMS, the website had several common problems.

### 1. Every Page Required Development Work

If the business needed a new SEO page, a developer had to create it manually.

Example:

```txt
/used-iron-beam
/steel-price
/iron-beam-factory
/buy-used-profile
```

Each page required a frontend route and usually duplicated layout logic.

---

### 2. SEO Execution Was Slow

SEO teams often need to publish pages quickly.

They may need pages for:

* search keywords,
* product categories,
* local landing pages,
* campaign pages,
* service pages,
* comparison pages.

Without a CMS, SEO execution becomes dependent on development cycles.

---

### 3. Repeated Frontend Code

Many pages share similar structures:

* hero section,
* intro content,
* feature list,
* FAQ,
* call-to-action,
* related products,
* price table.

But without a Page CMS, these sections are often recreated repeatedly.

---

### 4. Admins Could Not Manage Pages

Admins could update some content, but they could not create complete pages.

They needed developer support for:

* creating a page,
* changing the URL slug,
* editing meta tags,
* adding sections,
* reordering sections,
* publishing or archiving a page.

---

### 5. Website Scaling Became Harder

When the number of pages grows, hardcoded pages become difficult to maintain.

The better solution is to make pages data-driven.

---

## The Goal

The main goal was to build a CMS where admins can create and manage pages from the dashboard.

The system should support:

* dynamic page creation,
* custom URL slug,
* SEO fields,
* page status,
* reusable sections,
* section ordering,
* draft and publish workflow,
* preview support,
* frontend dynamic rendering,
* scalable backend architecture.

The admin should be able to create a page like this:

```json
{
  "title": "Used Iron Beam",
  "h1": "Buy Used Iron Beam with Daily Price",
  "urlSlug": "used-iron-beam",
  "pageType": "seo",
  "status": "published",
  "sections": [
    {
      "type": "hero",
      "title": "Used Iron Beam",
      "subtitle": "Daily price inquiry and expert consultation",
      "order": 1,
      "isActive": true
    },
    {
      "type": "content",
      "title": "What should you check before buying used iron beams?",
      "content": "<p>Before buying used iron beams, you should check quality, rust level, weight, and standard.</p>",
      "order": 2,
      "isActive": true
    },
    {
      "type": "faq",
      "title": "Frequently Asked Questions",
      "order": 3,
      "isActive": true
    }
  ]
}
```

Then the frontend should render this page automatically from the slug:

```txt
/used-iron-beam
```

---

## System Overview

The system has two main parts:

```txt
Admin Dashboard
      |
      | Create / Update / Publish Page
      v
Backend Page CMS API
      |
      | Store structured page data
      v
Database
      |
      | Get published page by slug
      v
Frontend Dynamic Renderer
      |
      | Render sections dynamically
      v
Public Website Page
```

The backend is responsible for:

* validating page data,
* generating and validating slugs,
* storing page content,
* managing publication status,
* exposing public page APIs.

The frontend is responsible for:

* reading the slug,
* fetching the published page,
* mapping section types to React components,
* rendering the final page.

---

## Page CMS Core Features

A production-ready Page CMS should support these features:

| Feature                  | Description                                                  |
| ------------------------ | ------------------------------------------------------------ |
| Dynamic Pages            | Admin can create pages without writing code                  |
| Custom Slug              | Each page has a unique URL                                   |
| SEO Fields               | Meta title, meta description, canonical URL, Open Graph data |
| Sections                 | Pages are built from reusable sections                       |
| Section Ordering         | Admin can reorder page sections                              |
| Draft / Published Status | Pages can be saved as draft before publishing                |
| Preview Mode             | Admin can preview before publishing                          |
| Archive Support          | Old pages can be archived without deletion                   |
| Safe Rendering           | HTML content should be sanitized                             |
| Server-Side Rendering    | SEO pages should be rendered server-side                     |

---

## Folder Structure

A clean backend structure can be designed using a layered architecture.

```txt
src/modules/page/
│
├── domain/
│   ├── entities/
│   │   ├── Page.ts
│   │   └── PageSection.ts
│   │
│   ├── repositories/
│   │   └── PageRepository.ts
│   │
│   └── services/
│       └── PageSlugService.ts
│
├── application/
│   ├── dto/
│   │   ├── CreatePageDto.ts
│   │   ├── UpdatePageDto.ts
│   │   └── PageFiltersDto.ts
│   │
│   ├── usecases/
│   │   ├── CreatePageUseCase.ts
│   │   ├── UpdatePageUseCase.ts
│   │   ├── GetPageByIdUseCase.ts
│   │   ├── GetPublishedPageBySlugUseCase.ts
│   │   ├── ListPagesUseCase.ts
│   │   └── DeletePageUseCase.ts
│   │
│   └── services/
│       └── PageService.ts
│
├── infrastructure/
│   ├── database/
│   │   └── mongoose/
│   │       ├── PageModel.ts
│   │       └── MongoPageRepository.ts
│   │
│   └── di/
│       └── PageModule.ts
│
└── presentation/
    ├── controllers/
    │   └── PageController.ts
    │
    ├── routes/
    │   └── page.routes.ts
    │
    └── validators/
        └── page.validator.ts
```

This structure separates:

* business rules,
* application use cases,
* database implementation,
* HTTP layer.

That makes the module easier to test, maintain, and extend.

---

## Page Domain Model

The `Page` entity represents the main page structure.

```ts
// src/modules/page/domain/entities/Page.ts

import { PageSection } from "./PageSection";

export type PageStatus = "draft" | "published" | "archived";

export type PageType =
  | "seo"
  | "landing"
  | "service"
  | "category"
  | "custom";

export interface PageSeo {
  metaTitle?: string;
  metaDescription?: string;
  canonicalUrl?: string;
  noIndex?: boolean;
  ogTitle?: string;
  ogDescription?: string;
  ogImage?: string;
}

export interface PageProps {
  id?: string;

  title: string;
  h1: string;
  urlSlug: string;
  pageType: PageType;
  status: PageStatus;

  introContent?: string;
  content?: string;

  seo?: PageSeo;

  sections?: PageSection[];

  createdAt?: Date;
  updatedAt?: Date;
}

export class Page {
  public readonly id?: string;

  public title: string;
  public h1: string;
  public urlSlug: string;
  public pageType: PageType;
  public status: PageStatus;

  public introContent?: string;
  public content?: string;

  public seo: PageSeo;
  public sections: PageSection[];

  public readonly createdAt?: Date;
  public readonly updatedAt?: Date;

  constructor(props: PageProps) {
    this.id = props.id;

    this.title = props.title;
    this.h1 = props.h1;
    this.urlSlug = props.urlSlug;
    this.pageType = props.pageType;
    this.status = props.status;

    this.introContent = props.introContent;
    this.content = props.content;

    this.seo = props.seo ?? {};
    this.sections = props.sections ?? [];

    this.createdAt = props.createdAt;
    this.updatedAt = props.updatedAt;
  }

  publish() {
    this.status = "published";
  }

  archive() {
    this.status = "archived";
  }

  saveAsDraft() {
    this.status = "draft";
  }

  isPublished() {
    return this.status === "published";
  }
}
```

The entity contains simple business behavior:

```ts
page.publish();
page.archive();
page.saveAsDraft();
page.isPublished();
```

This is better than spreading status-changing logic across controllers.

---

## Page Section Model

Each page can have multiple sections.

```ts
// src/modules/page/domain/entities/PageSection.ts

export type PageSectionType =
  | "hero"
  | "content"
  | "imageText"
  | "features"
  | "faq"
  | "cta"
  | "productList"
  | "priceTable"
  | "video";

export interface PageSectionButton {
  text: string;
  url: string;
  variant?: "primary" | "secondary" | "outline";
}

export interface PageSectionItem {
  title?: string;
  description?: string;
  icon?: string;
  imageUrl?: string;
  url?: string;
}

export interface PageSection {
  id?: string;

  type: PageSectionType;

  title?: string;
  subtitle?: string;
  description?: string;
  content?: string;

  imageUrl?: string;
  imageAlt?: string;

  videoUrl?: string;

  button?: PageSectionButton;

  items?: PageSectionItem[];

  order: number;
  isActive: boolean;
}
```

This model allows the CMS to support different page structures.

For example, a landing page can have:

```txt
Hero -> Features -> CTA -> FAQ
```

An SEO page can have:

```txt
Hero -> Content -> Price Table -> FAQ -> CTA
```

A service page can have:

```txt
Hero -> Image + Text -> Features -> Contact CTA
```

---

## DTOs

DTOs define the input structure for application use cases.

### Create Page DTO

```ts
// src/modules/page/application/dto/CreatePageDto.ts

import {
  PageSeo,
  PageStatus,
  PageType,
} from "../../domain/entities/Page";

import { PageSection } from "../../domain/entities/PageSection";

export interface CreatePageDto {
  title: string;
  h1: string;
  urlSlug?: string;
  pageType: PageType;
  status?: PageStatus;

  introContent?: string;
  content?: string;

  seo?: PageSeo;

  sections?: PageSection[];
}
```

### Update Page DTO

```ts
// src/modules/page/application/dto/UpdatePageDto.ts

import {
  PageSeo,
  PageStatus,
  PageType,
} from "../../domain/entities/Page";

import { PageSection } from "../../domain/entities/PageSection";

export interface UpdatePageDto {
  title?: string;
  h1?: string;
  urlSlug?: string;
  pageType?: PageType;
  status?: PageStatus;

  introContent?: string;
  content?: string;

  seo?: PageSeo;

  sections?: PageSection[];
}
```

### Page Filters DTO

```ts
// src/modules/page/application/dto/PageFiltersDto.ts

import { PageStatus, PageType } from "../../domain/entities/Page";

export interface PageFiltersDto {
  search?: string;
  status?: PageStatus;
  pageType?: PageType;

  page?: number;
  limit?: number;
}
```

---

## Repository Contract

The repository contract belongs to the domain layer.

It defines what the application needs, not how the database works.

```ts
// src/modules/page/domain/repositories/PageRepository.ts

import { Page } from "../entities/Page";
import { PageFiltersDto } from "../../application/dto/PageFiltersDto";

export interface PaginatedPages {
  data: Page[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
}

export interface PageRepository {
  create(page: Page): Promise<Page>;

  update(id: string, page: Partial<Page>): Promise<Page | null>;

  findById(id: string): Promise<Page | null>;

  findBySlug(urlSlug: string): Promise<Page | null>;

  findPublishedBySlug(urlSlug: string): Promise<Page | null>;

  findMany(filters: PageFiltersDto): Promise<PaginatedPages>;

  delete(id: string): Promise<boolean>;

  slugExists(urlSlug: string, excludeId?: string): Promise<boolean>;
}
```

This makes it possible to replace MongoDB with another database later without changing use cases.

---

## Slug Service

The slug service creates safe and readable URL slugs.

```ts
// src/modules/page/domain/services/PageSlugService.ts

export class PageSlugService {
  normalize(value: string): string {
    return value
      .toLowerCase()
      .trim()
      .replace(/['"]/g, "")
      .replace(/[^a-z0-9\u0600-\u06FF]+/g, "-")
      .replace(/^-+|-+$/g, "");
  }

  generateFromTitle(title: string): string {
    return this.normalize(title);
  }

  validate(slug: string): void {
    if (!slug || slug.trim().length < 2) {
      throw new Error("Page slug must contain at least 2 characters.");
    }

    if (slug.length > 120) {
      throw new Error("Page slug cannot be longer than 120 characters.");
    }

    const validPattern = /^[a-z0-9\u0600-\u06FF]+(?:-[a-z0-9\u0600-\u06FF]+)*$/;

    if (!validPattern.test(slug)) {
      throw new Error("Page slug contains invalid characters.");
    }
  }
}
```

Example:

```ts
const slugService = new PageSlugService();

slugService.generateFromTitle("Used Iron Beam");
// used-iron-beam

slugService.generateFromTitle("خرید تیرآهن دست دوم");
// خرید-تیرآهن-دست-دوم
```

---

## Use Cases

Use cases contain the application business flow.

They do not know about Express, MongoDB, or HTTP.

---

## Create Page Use Case

```ts
// src/modules/page/application/usecases/CreatePageUseCase.ts

import { Page } from "../../domain/entities/Page";
import { PageRepository } from "../../domain/repositories/PageRepository";
import { PageSlugService } from "../../domain/services/PageSlugService";
import { CreatePageDto } from "../dto/CreatePageDto";

export class CreatePageUseCase {
  constructor(
    private readonly pageRepository: PageRepository,
    private readonly pageSlugService: PageSlugService
  ) {}

  async execute(dto: CreatePageDto): Promise<Page> {
    const rawSlug = dto.urlSlug?.trim()
      ? dto.urlSlug
      : this.pageSlugService.generateFromTitle(dto.title);

    const normalizedSlug = this.pageSlugService.normalize(rawSlug);

    this.pageSlugService.validate(normalizedSlug);

    const exists = await this.pageRepository.slugExists(normalizedSlug);

    if (exists) {
      throw new Error("A page with this slug already exists.");
    }

    const page = new Page({
      title: dto.title,
      h1: dto.h1,
      urlSlug: normalizedSlug,
      pageType: dto.pageType,
      status: dto.status ?? "draft",
      introContent: dto.introContent,
      content: dto.content,
      seo: dto.seo,
      sections: dto.sections ?? [],
    });

    return this.pageRepository.create(page);
  }
}
```

This use case handles:

* slug generation,
* slug normalization,
* slug validation,
* duplicate slug prevention,
* default draft status.

---

## Update Page Use Case

```ts
// src/modules/page/application/usecases/UpdatePageUseCase.ts

import { Page } from "../../domain/entities/Page";
import { PageRepository } from "../../domain/repositories/PageRepository";
import { PageSlugService } from "../../domain/services/PageSlugService";
import { UpdatePageDto } from "../dto/UpdatePageDto";

export class UpdatePageUseCase {
  constructor(
    private readonly pageRepository: PageRepository,
    private readonly pageSlugService: PageSlugService
  ) {}

  async execute(id: string, dto: UpdatePageDto): Promise<Page> {
    const currentPage = await this.pageRepository.findById(id);

    if (!currentPage) {
      throw new Error("Page not found.");
    }

    let normalizedSlug = dto.urlSlug;

    if (dto.urlSlug) {
      normalizedSlug = this.pageSlugService.normalize(dto.urlSlug);

      this.pageSlugService.validate(normalizedSlug);

      const exists = await this.pageRepository.slugExists(normalizedSlug, id);

      if (exists) {
        throw new Error("A page with this slug already exists.");
      }
    }

    const updatedPage = await this.pageRepository.update(id, {
      ...dto,
      urlSlug: normalizedSlug,
    });

    if (!updatedPage) {
      throw new Error("Failed to update page.");
    }

    return updatedPage;
  }
}
```

This prevents two pages from using the same slug.

---

## Get Published Page By Slug Use Case

This use case is used by the public website.

```ts
// src/modules/page/application/usecases/GetPublishedPageBySlugUseCase.ts

import { Page } from "../../domain/entities/Page";
import { PageRepository } from "../../domain/repositories/PageRepository";

export class GetPublishedPageBySlugUseCase {
  constructor(private readonly pageRepository: PageRepository) {}

  async execute(slug: string): Promise<Page> {
    const page = await this.pageRepository.findPublishedBySlug(slug);

    if (!page) {
      throw new Error("Published page not found.");
    }

    return page;
  }
}
```

This is important because draft pages should not be visible publicly.

---

## Get Page By ID Use Case

```ts
// src/modules/page/application/usecases/GetPageByIdUseCase.ts

import { Page } from "../../domain/entities/Page";
import { PageRepository } from "../../domain/repositories/PageRepository";

export class GetPageByIdUseCase {
  constructor(private readonly pageRepository: PageRepository) {}

  async execute(id: string): Promise<Page> {
    const page = await this.pageRepository.findById(id);

    if (!page) {
      throw new Error("Page not found.");
    }

    return page;
  }
}
```

---

## List Pages Use Case

```ts
// src/modules/page/application/usecases/ListPagesUseCase.ts

import {
  PageRepository,
  PaginatedPages,
} from "../../domain/repositories/PageRepository";

import { PageFiltersDto } from "../dto/PageFiltersDto";

export class ListPagesUseCase {
  constructor(private readonly pageRepository: PageRepository) {}

  async execute(filters: PageFiltersDto): Promise<PaginatedPages> {
    const page = Math.max(Number(filters.page ?? 1), 1);
    const limit = Math.min(Math.max(Number(filters.limit ?? 10), 1), 100);

    return this.pageRepository.findMany({
      ...filters,
      page,
      limit,
    });
  }
}
```

---

## Delete Page Use Case

```ts
// src/modules/page/application/usecases/DeletePageUseCase.ts

import { PageRepository } from "../../domain/repositories/PageRepository";

export class DeletePageUseCase {
  constructor(private readonly pageRepository: PageRepository) {}

  async execute(id: string): Promise<boolean> {
    const page = await this.pageRepository.findById(id);

    if (!page) {
      throw new Error("Page not found.");
    }

    return this.pageRepository.delete(id);
  }
}
```

In many production systems, I prefer soft delete or archive instead of physical delete.

For example:

```ts
page.archive();
```

This avoids broken URLs and accidental content loss.

---

## Application Service

The service acts as a clean facade for controllers.

```ts
// src/modules/page/application/services/PageService.ts

import { CreatePageDto } from "../dto/CreatePageDto";
import { UpdatePageDto } from "../dto/UpdatePageDto";
import { PageFiltersDto } from "../dto/PageFiltersDto";

import { CreatePageUseCase } from "../usecases/CreatePageUseCase";
import { UpdatePageUseCase } from "../usecases/UpdatePageUseCase";
import { GetPageByIdUseCase } from "../usecases/GetPageByIdUseCase";
import { GetPublishedPageBySlugUseCase } from "../usecases/GetPublishedPageBySlugUseCase";
import { ListPagesUseCase } from "../usecases/ListPagesUseCase";
import { DeletePageUseCase } from "../usecases/DeletePageUseCase";

export class PageService {
  constructor(
    private readonly createPageUseCase: CreatePageUseCase,
    private readonly updatePageUseCase: UpdatePageUseCase,
    private readonly getPageByIdUseCase: GetPageByIdUseCase,
    private readonly getPublishedPageBySlugUseCase: GetPublishedPageBySlugUseCase,
    private readonly listPagesUseCase: ListPagesUseCase,
    private readonly deletePageUseCase: DeletePageUseCase
  ) {}

  create(dto: CreatePageDto) {
    return this.createPageUseCase.execute(dto);
  }

  update(id: string, dto: UpdatePageDto) {
    return this.updatePageUseCase.execute(id, dto);
  }

  getById(id: string) {
    return this.getPageByIdUseCase.execute(id);
  }

  getPublishedBySlug(slug: string) {
    return this.getPublishedPageBySlugUseCase.execute(slug);
  }

  list(filters: PageFiltersDto) {
    return this.listPagesUseCase.execute(filters);
  }

  delete(id: string) {
    return this.deletePageUseCase.execute(id);
  }
}
```

The controller should not contain business logic.

It should only:

* read request data,
* call service methods,
* return HTTP responses.

---

## MongoDB Schema

MongoDB works well for Page CMS because each page can contain nested flexible sections.

```ts
// src/modules/page/infrastructure/database/mongoose/PageModel.ts

import mongoose, { Schema, Document, Model } from "mongoose";

export type PageDocument = Document & {
  title: string;
  h1: string;
  urlSlug: string;
  pageType: string;
  status: string;

  introContent?: string;
  content?: string;

  seo?: {
    metaTitle?: string;
    metaDescription?: string;
    canonicalUrl?: string;
    noIndex?: boolean;
    ogTitle?: string;
    ogDescription?: string;
    ogImage?: string;
  };

  sections?: Array<{
    type: string;

    title?: string;
    subtitle?: string;
    description?: string;
    content?: string;

    imageUrl?: string;
    imageAlt?: string;

    videoUrl?: string;

    button?: {
      text: string;
      url: string;
      variant?: string;
    };

    items?: Array<{
      title?: string;
      description?: string;
      icon?: string;
      imageUrl?: string;
      url?: string;
    }>;

    order: number;
    isActive: boolean;
  }>;

  createdAt: Date;
  updatedAt: Date;
};

const PageSectionItemSchema = new Schema(
  {
    title: String,
    description: String,
    icon: String,
    imageUrl: String,
    url: String,
  },
  { _id: false }
);

const PageSectionButtonSchema = new Schema(
  {
    text: { type: String, required: true },
    url: { type: String, required: true },
    variant: {
      type: String,
      enum: ["primary", "secondary", "outline"],
      default: "primary",
    },
  },
  { _id: false }
);

const PageSectionSchema = new Schema(
  {
    type: {
      type: String,
      required: true,
      enum: [
        "hero",
        "content",
        "imageText",
        "features",
        "faq",
        "cta",
        "productList",
        "priceTable",
        "video",
      ],
    },

    title: String,
    subtitle: String,
    description: String,
    content: String,

    imageUrl: String,
    imageAlt: String,

    videoUrl: String,

    button: PageSectionButtonSchema,

    items: [PageSectionItemSchema],

    order: {
      type: Number,
      required: true,
      default: 1,
    },

    isActive: {
      type: Boolean,
      default: true,
    },
  },
  { _id: true }
);

const PageSeoSchema = new Schema(
  {
    metaTitle: String,
    metaDescription: String,
    canonicalUrl: String,
    noIndex: {
      type: Boolean,
      default: false,
    },
    ogTitle: String,
    ogDescription: String,
    ogImage: String,
  },
  { _id: false }
);

const PageSchema = new Schema<PageDocument>(
  {
    title: {
      type: String,
      required: true,
      trim: true,
    },

    h1: {
      type: String,
      required: true,
      trim: true,
    },

    urlSlug: {
      type: String,
      required: true,
      unique: true,
      index: true,
      trim: true,
    },

    pageType: {
      type: String,
      required: true,
      enum: ["seo", "landing", "service", "category", "custom"],
      default: "custom",
    },

    status: {
      type: String,
      required: true,
      enum: ["draft", "published", "archived"],
      default: "draft",
      index: true,
    },

    introContent: String,

    content: String,

    seo: PageSeoSchema,

    sections: [PageSectionSchema],
  },
  {
    timestamps: true,
  }
);

PageSchema.index({ status: 1, urlSlug: 1 });
PageSchema.index({ title: "text", h1: "text", introContent: "text" });

export const PageModel: Model<PageDocument> =
  mongoose.models.Page || mongoose.model<PageDocument>("Page", PageSchema);
```

Important indexes:

```ts
PageSchema.index({ status: 1, urlSlug: 1 });
PageSchema.index({ title: "text", h1: "text", introContent: "text" });
```

These help with:

* public lookup by slug,
* admin search,
* filtering by status.

---

## Mongo Repository Implementation

```ts
// src/modules/page/infrastructure/database/mongoose/MongoPageRepository.ts

import { Page } from "../../../domain/entities/Page";
import {
  PageRepository,
  PaginatedPages,
} from "../../../domain/repositories/PageRepository";

import { PageFiltersDto } from "../../../application/dto/PageFiltersDto";
import { PageModel, PageDocument } from "./PageModel";

export class MongoPageRepository implements PageRepository {
  private toDomain(document: PageDocument): Page {
    return new Page({
      id: String(document._id),
      title: document.title,
      h1: document.h1,
      urlSlug: document.urlSlug,
      pageType: document.pageType as Page["pageType"],
      status: document.status as Page["status"],
      introContent: document.introContent,
      content: document.content,
      seo: document.seo,
      sections: document.sections?.map((section: any) => ({
        id: String(section._id),
        type: section.type,
        title: section.title,
        subtitle: section.subtitle,
        description: section.description,
        content: section.content,
        imageUrl: section.imageUrl,
        imageAlt: section.imageAlt,
        videoUrl: section.videoUrl,
        button: section.button,
        items: section.items,
        order: section.order,
        isActive: section.isActive,
      })),
      createdAt: document.createdAt,
      updatedAt: document.updatedAt,
    });
  }

  async create(page: Page): Promise<Page> {
    const created = await PageModel.create({
      title: page.title,
      h1: page.h1,
      urlSlug: page.urlSlug,
      pageType: page.pageType,
      status: page.status,
      introContent: page.introContent,
      content: page.content,
      seo: page.seo,
      sections: page.sections,
    });

    return this.toDomain(created);
  }

  async update(id: string, page: Partial<Page>): Promise<Page | null> {
    const updated = await PageModel.findByIdAndUpdate(
      id,
      {
        $set: {
          ...page,
        },
      },
      {
        new: true,
        runValidators: true,
      }
    );

    return updated ? this.toDomain(updated) : null;
  }

  async findById(id: string): Promise<Page | null> {
    const page = await PageModel.findById(id);

    return page ? this.toDomain(page) : null;
  }

  async findBySlug(urlSlug: string): Promise<Page | null> {
    const page = await PageModel.findOne({ urlSlug });

    return page ? this.toDomain(page) : null;
  }

  async findPublishedBySlug(urlSlug: string): Promise<Page | null> {
    const page = await PageModel.findOne({
      urlSlug,
      status: "published",
    });

    return page ? this.toDomain(page) : null;
  }

  async findMany(filters: PageFiltersDto): Promise<PaginatedPages> {
    const page = Number(filters.page ?? 1);
    const limit = Number(filters.limit ?? 10);
    const skip = (page - 1) * limit;

    const query: Record<string, unknown> = {};

    if (filters.status) {
      query.status = filters.status;
    }

    if (filters.pageType) {
      query.pageType = filters.pageType;
    }

    if (filters.search?.trim()) {
      query.$or = [
        { title: { $regex: filters.search, $options: "i" } },
        { h1: { $regex: filters.search, $options: "i" } },
        { urlSlug: { $regex: filters.search, $options: "i" } },
      ];
    }

    const [data, total] = await Promise.all([
      PageModel.find(query)
        .sort({ updatedAt: -1 })
        .skip(skip)
        .limit(limit),
      PageModel.countDocuments(query),
    ]);

    return {
      data: data.map((item) => this.toDomain(item)),
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
    };
  }

  async delete(id: string): Promise<boolean> {
    const result = await PageModel.findByIdAndDelete(id);

    return Boolean(result);
  }

  async slugExists(urlSlug: string, excludeId?: string): Promise<boolean> {
    const query: Record<string, unknown> = { urlSlug };

    if (excludeId) {
      query._id = { $ne: excludeId };
    }

    const page = await PageModel.exists(query);

    return Boolean(page);
  }
}
```

---

## Express Controller

```ts
// src/modules/page/presentation/controllers/PageController.ts

import { Request, Response, NextFunction } from "express";
import { PageService } from "../../application/services/PageService";

export class PageController {
  constructor(private readonly pageService: PageService) {}

  create = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const page = await this.pageService.create(req.body);

      return res.status(201).json({
        message: "Page created successfully.",
        data: page,
      });
    } catch (error) {
      next(error);
    }
  };

  update = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const page = await this.pageService.update(req.params.id, req.body);

      return res.status(200).json({
        message: "Page updated successfully.",
        data: page,
      });
    } catch (error) {
      next(error);
    }
  };

  getById = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const page = await this.pageService.getById(req.params.id);

      return res.status(200).json({
        message: "Page fetched successfully.",
        data: page,
      });
    } catch (error) {
      next(error);
    }
  };

  getPublishedBySlug = async (
    req: Request,
    res: Response,
    next: NextFunction
  ) => {
    try {
      const page = await this.pageService.getPublishedBySlug(req.params.slug);

      return res.status(200).json({
        message: "Published page fetched successfully.",
        data: page,
      });
    } catch (error) {
      next(error);
    }
  };

  list = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const result = await this.pageService.list({
        search: req.query.search as string,
        status: req.query.status as any,
        pageType: req.query.pageType as any,
        page: Number(req.query.page ?? 1),
        limit: Number(req.query.limit ?? 10),
      });

      return res.status(200).json({
        message: "Pages fetched successfully.",
        ...result,
      });
    } catch (error) {
      next(error);
    }
  };

  delete = async (req: Request, res: Response, next: NextFunction) => {
    try {
      await this.pageService.delete(req.params.id);

      return res.status(200).json({
        message: "Page deleted successfully.",
      });
    } catch (error) {
      next(error);
    }
  };
}
```

The controller is intentionally thin.

Business rules stay inside use cases.

---

## Routes

```ts
// src/modules/page/presentation/routes/page.routes.ts

import { Router } from "express";
import { PageController } from "../controllers/PageController";

export function createPageRoutes(pageController: PageController): Router {
  const router = Router();

  router.post("/", pageController.create);

  router.get("/", pageController.list);

  router.get("/published/:slug", pageController.getPublishedBySlug);

  router.get("/:id", pageController.getById);

  router.patch("/:id", pageController.update);

  router.delete("/:id", pageController.delete);

  return router;
}
```

Example endpoints:

```txt
POST   /api/pages
GET    /api/pages
GET    /api/pages/:id
PATCH  /api/pages/:id
DELETE /api/pages/:id
GET    /api/pages/published/:slug
```

---

## Dependency Injection Module

```ts
// src/modules/page/infrastructure/di/PageModule.ts

import { MongoPageRepository } from "../database/mongoose/MongoPageRepository";

import { PageSlugService } from "../../domain/services/PageSlugService";

import { CreatePageUseCase } from "../../application/usecases/CreatePageUseCase";
import { UpdatePageUseCase } from "../../application/usecases/UpdatePageUseCase";
import { GetPageByIdUseCase } from "../../application/usecases/GetPageByIdUseCase";
import { GetPublishedPageBySlugUseCase } from "../../application/usecases/GetPublishedPageBySlugUseCase";
import { ListPagesUseCase } from "../../application/usecases/ListPagesUseCase";
import { DeletePageUseCase } from "../../application/usecases/DeletePageUseCase";

import { PageService } from "../../application/services/PageService";

import { PageController } from "../../presentation/controllers/PageController";
import { createPageRoutes } from "../../presentation/routes/page.routes";

export class PageModule {
  public readonly routes;

  constructor() {
    const pageRepository = new MongoPageRepository();

    const pageSlugService = new PageSlugService();

    const createPageUseCase = new CreatePageUseCase(
      pageRepository,
      pageSlugService
    );

    const updatePageUseCase = new UpdatePageUseCase(
      pageRepository,
      pageSlugService
    );

    const getPageByIdUseCase = new GetPageByIdUseCase(pageRepository);

    const getPublishedPageBySlugUseCase =
      new GetPublishedPageBySlugUseCase(pageRepository);

    const listPagesUseCase = new ListPagesUseCase(pageRepository);

    const deletePageUseCase = new DeletePageUseCase(pageRepository);

    const pageService = new PageService(
      createPageUseCase,
      updatePageUseCase,
      getPageByIdUseCase,
      getPublishedPageBySlugUseCase,
      listPagesUseCase,
      deletePageUseCase
    );

    const pageController = new PageController(pageService);

    this.routes = createPageRoutes(pageController);
  }
}
```

Register it in the main Express app:

```ts
import express from "express";
import { PageModule } from "./modules/page/infrastructure/di/PageModule";

const app = express();

app.use(express.json());

const pageModule = new PageModule();

app.use("/api/pages", pageModule.routes);
```

---

## Frontend Dynamic Rendering

On the frontend, we can create one dynamic route.

Example using Next.js App Router:

```txt
app/[slug]/page.tsx
```

```tsx
// app/[slug]/page.tsx

import { notFound } from "next/navigation";
import { RenderPageSections } from "@/features/page/components/RenderPageSections";

interface PageProps {
  params: {
    slug: string;
  };
}

async function getPageBySlug(slug: string) {
  const response = await fetch(
    `${process.env.NEXT_PUBLIC_API_URL}/api/pages/published/${slug}`,
    {
      cache: "no-store",
    }
  );

  if (!response.ok) {
    return null;
  }

  const result = await response.json();

  return result.data;
}

export default async function DynamicPage({ params }: PageProps) {
  const page = await getPageBySlug(params.slug);

  if (!page) {
    notFound();
  }

  return (
    <main>
      <h1>{page.h1}</h1>

      {page.introContent ? <p>{page.introContent}</p> : null}

      <RenderPageSections sections={page.sections} />
    </main>
  );
}
```

---

## Section Renderer

```tsx
// features/page/components/RenderPageSections.tsx

import { HeroSection } from "./sections/HeroSection";
import { ContentSection } from "./sections/ContentSection";
import { ImageTextSection } from "./sections/ImageTextSection";
import { FeaturesSection } from "./sections/FeaturesSection";
import { FaqSection } from "./sections/FaqSection";
import { CtaSection } from "./sections/CtaSection";
import { VideoSection } from "./sections/VideoSection";

const sectionMap = {
  hero: HeroSection,
  content: ContentSection,
  imageText: ImageTextSection,
  features: FeaturesSection,
  faq: FaqSection,
  cta: CtaSection,
  video: VideoSection,
};

interface RenderPageSectionsProps {
  sections: Array<{
    id: string;
    type: keyof typeof sectionMap;
    order: number;
    isActive: boolean;
    [key: string]: unknown;
  }>;
}

export function RenderPageSections({ sections }: RenderPageSectionsProps) {
  return (
    <>
      {sections
        .filter((section) => section.isActive)
        .sort((a, b) => a.order - b.order)
        .map((section) => {
          const Component = sectionMap[section.type];

          if (!Component) {
            return null;
          }

          return <Component key={section.id} section={section} />;
        })}
    </>
  );
}
```

---

## Example Section Components

### Hero Section

```tsx
// features/page/components/sections/HeroSection.tsx

interface HeroSectionProps {
  section: {
    title?: string;
    subtitle?: string;
    description?: string;
    imageUrl?: string;
    imageAlt?: string;
    button?: {
      text: string;
      url: string;
    };
  };
}

export function HeroSection({ section }: HeroSectionProps) {
  return (
    <section>
      <div>
        {section.title ? <h2>{section.title}</h2> : null}

        {section.subtitle ? <h3>{section.subtitle}</h3> : null}

        {section.description ? <p>{section.description}</p> : null}

        {section.button ? (
          <a href={section.button.url}>{section.button.text}</a>
        ) : null}
      </div>

      {section.imageUrl ? (
        <img src={section.imageUrl} alt={section.imageAlt ?? section.title ?? ""} />
      ) : null}
    </section>
  );
}
```

### Content Section

```tsx
// features/page/components/sections/ContentSection.tsx

interface ContentSectionProps {
  section: {
    title?: string;
    content?: string;
  };
}

export function ContentSection({ section }: ContentSectionProps) {
  return (
    <section>
      {section.title ? <h2>{section.title}</h2> : null}

      {section.content ? (
        <div dangerouslySetInnerHTML={{ __html: section.content }} />
      ) : null}
    </section>
  );
}
```

Important:

If you render HTML using `dangerouslySetInnerHTML`, sanitize the content before saving or before rendering.

---

## SEO Metadata in Next.js

The page data can also generate dynamic SEO metadata.

```tsx
// app/[slug]/page.tsx

import { Metadata } from "next";

interface PageProps {
  params: {
    slug: string;
  };
}

async function getPageBySlug(slug: string) {
  const response = await fetch(
    `${process.env.NEXT_PUBLIC_API_URL}/api/pages/published/${slug}`,
    {
      cache: "no-store",
    }
  );

  if (!response.ok) {
    return null;
  }

  const result = await response.json();

  return result.data;
}

export async function generateMetadata({
  params,
}: PageProps): Promise<Metadata> {
  const page = await getPageBySlug(params.slug);

  if (!page) {
    return {};
  }

  return {
    title: page.seo?.metaTitle ?? page.title,
    description: page.seo?.metaDescription ?? page.introContent,
    alternates: page.seo?.canonicalUrl
      ? {
          canonical: page.seo.canonicalUrl,
        }
      : undefined,
    robots: page.seo?.noIndex
      ? {
          index: false,
          follow: false,
        }
      : {
          index: true,
          follow: true,
        },
    openGraph: {
      title: page.seo?.ogTitle ?? page.seo?.metaTitle ?? page.title,
      description:
        page.seo?.ogDescription ??
        page.seo?.metaDescription ??
        page.introContent,
      images: page.seo?.ogImage ? [page.seo.ogImage] : [],
    },
  };
}
```

This gives SEO teams control over:

* meta title,
* meta description,
* canonical URL,
* Open Graph image,
* index/no-index behavior.

---

## API Flow

### Create Page

```txt
Admin Dashboard
      |
      | POST /api/pages
      v
PageController.create
      |
      v
PageService.create
      |
      v
CreatePageUseCase.execute
      |
      | validate slug
      | check duplicate slug
      | create Page entity
      v
PageRepository.create
      |
      v
MongoDB
```

---

### Public Page Rendering

```txt
User visits /used-iron-beam
      |
      v
Next.js dynamic route receives slug
      |
      v
GET /api/pages/published/used-iron-beam
      |
      v
GetPublishedPageBySlugUseCase
      |
      | only status = published
      v
MongoDB
      |
      v
Frontend renders sections dynamically
```

---

## Example Page Payload

```json
{
  "title": "Used Iron Beam",
  "h1": "Buy Used Iron Beam with Daily Price",
  "urlSlug": "used-iron-beam",
  "pageType": "seo",
  "status": "published",
  "introContent": "Find high-quality used iron beams at the best price.",
  "seo": {
    "metaTitle": "Buy Used Iron Beam | Best Daily Price",
    "metaDescription": "Buy high-quality used iron beams with expert consultation and daily price updates.",
    "canonicalUrl": "https://example.com/used-iron-beam",
    "noIndex": false,
    "ogTitle": "Used Iron Beam",
    "ogDescription": "Daily price inquiry for used iron beams.",
    "ogImage": "https://example.com/images/used-iron-beam.jpg"
  },
  "sections": [
    {
      "type": "hero",
      "title": "Used Iron Beam",
      "subtitle": "Daily price inquiry and expert consultation",
      "description": "Buy reliable used iron beams for construction and industrial projects.",
      "imageUrl": "/uploads/pages/used-iron-beam.webp",
      "imageAlt": "Used iron beam",
      "button": {
        "text": "Get a Quote",
        "url": "/contact",
        "variant": "primary"
      },
      "order": 1,
      "isActive": true
    },
    {
      "type": "content",
      "title": "What should you check before buying used iron beams?",
      "content": "<p>Before buying used iron beams, you should check quality, rust level, weight, and product standard.</p>",
      "order": 2,
      "isActive": true
    },
    {
      "type": "faq",
      "title": "Frequently Asked Questions",
      "items": [
        {
          "title": "Is used iron beam suitable for construction?",
          "description": "It depends on quality, condition, weight, and engineering requirements."
        },
        {
          "title": "How is the price calculated?",
          "description": "The price usually depends on weight, condition, size, and market rate."
        }
      ],
      "order": 3,
      "isActive": true
    }
  ]
}
```

---

## Problems Solved

### 1. Reduced Developer Dependency

Admins can create pages without waiting for developers.

This reduces repeated work and speeds up content operations.

---

### 2. Faster SEO Publishing

SEO pages can be published quickly.

The SEO team can manage:

* slug,
* H1,
* meta title,
* meta description,
* canonical URL,
* content sections,
* FAQ content.

---

### 3. Reusable Frontend Components

Instead of building every page manually, the frontend uses reusable section components.

The CMS controls the data.

The frontend controls the design.

This creates a good balance between flexibility and maintainability.

---

### 4. Better Maintainability

The frontend does not need hundreds of hardcoded pages.

Most pages can be handled by one dynamic route and one section renderer.

---

### 5. Scalable Architecture

New section types can be added later.

For example:

```txt
testimonial
comparisonTable
downloadBox
relatedProducts
inquiryForm
mapSection
```

The database model and renderer can evolve with business needs.

---

### 6. Better Business Agility

Marketing teams can create pages for campaigns.

SEO teams can create keyword-based landing pages.

Admins can update content without deployment.

Developers can focus on core product features.

---

## Important Implementation Notes

### 1. Use Unique Slugs

Every page must have a unique slug.

```ts
const exists = await pageRepository.slugExists(normalizedSlug);

if (exists) {
  throw new Error("A page with this slug already exists.");
}
```

Without this rule, two pages may conflict on the same URL.

---

### 2. Keep Draft Pages Private

Public APIs should only return published pages.

```ts
await PageModel.findOne({
  urlSlug,
  status: "published",
});
```

Draft pages should only be visible in the admin panel or preview mode.

---

### 3. Sanitize HTML Content

If your CMS supports HTML content, sanitize it.

Never trust raw HTML from users.

Possible solutions:

* sanitize before saving,
* sanitize before rendering,
* restrict allowed tags,
* restrict allowed attributes.

---

### 4. Prefer Archive Over Delete

For content-heavy websites, deleting pages can break URLs.

A safer option is:

```ts
status: "archived"
```

This keeps history and prevents accidental data loss.

---

### 5. Use Server-Side Rendering for SEO Pages

For SEO pages, avoid relying only on client-side rendering.

Search engines should receive meaningful HTML from the server.

---

### 6. Validate Section Types

Only allow supported section types.

```ts
const allowedTypes = [
  "hero",
  "content",
  "imageText",
  "features",
  "faq",
  "cta",
  "productList",
  "priceTable",
  "video",
];

if (!allowedTypes.includes(section.type)) {
  throw new Error("Invalid section type.");
}
```

This prevents invalid data from breaking the frontend renderer.

---

### 7. Keep Frontend Rendering Controlled

The CMS should not control everything.

A good rule is:

> CMS controls content and structure.
> Frontend controls design and behavior.

This avoids turning the CMS into an unsafe page builder.

---

## Possible Future Improvements

The first version can be simple.

But later, the CMS can be improved with:

* page preview mode,
* version history,
* scheduled publishing,
* reusable section templates,
* drag-and-drop section ordering,
* role-based permissions,
* media library,
* internal linking suggestions,
* SEO score checker,
* schema.org structured data,
* multilingual pages,
* A/B testing support.

---

## Final Architecture Summary

```txt
Page CMS
│
├── Admin Dashboard
│   ├── Create Page
│   ├── Edit Page
│   ├── Manage SEO
│   ├── Manage Sections
│   ├── Preview Page
│   └── Publish Page
│
├── Backend API
│   ├── Controller
│   ├── Service
│   ├── Use Cases
│   ├── Repository Contract
│   └── MongoDB Repository
│
├── Database
│   ├── Page
│   ├── SEO Fields
│   └── Sections
│
└── Frontend
    ├── Dynamic Route
    ├── Fetch Page By Slug
    ├── Generate Metadata
    └── Render Sections
```

---

## Conclusion

A Dynamic Page CMS is not just a content feature.

It is a system design improvement.

It solves a real business and engineering problem:

* developers stop repeating similar pages,
* admins gain control over content,
* SEO teams move faster,
* the frontend becomes cleaner,
* the backend becomes a structured source of truth,
* the website becomes easier to scale.

The most important idea is simple:

> A page should not always be a hardcoded route.
> In many cases, a page should be a structured model that can be created, edited, published, and rendered dynamically.

This approach is especially useful for SEO-driven websites, landing pages, service pages, campaign pages, and business websites that need to grow quickly without increasing development overhead.
