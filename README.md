import dotenv from 'dotenv';
import * as fs from 'fs/promises';
import * as path from 'path';

import { PrismaRepository } from '@api/repository/repository.service';

dotenv.config();

export class LogsRscrm {
  private enable = false;
  private static readonly FILE_PATH = path.resolve(
    'src/api/integrations/channel/whatsapp/logs/logs.json'
  );

  // 🔥 fila em memória
  private buffer: any[] = [];
  private isFlushing = false;

  // 🔥 configuração
  private readonly MAX_BUFFER = 12; // quando atingir, já dispara flush
  private readonly FLUSH_INTERVAL_MS = 2000; // flush automático a cada 2s

  constructor(private readonly prisma?: PrismaRepository) {
    this.enable = process.env.RSCRM_LOGS_ENABLE === 'true';
    console.log('[LogsRscrm] enable:', this.enable);

    // 🔥 flush periódico automático
    if (this.enable) {
      setInterval(() => this.flush().catch(console.error), this.FLUSH_INTERVAL_MS);
    }
  }

  public async criarLog(log: any): Promise<void> {
    if (!this.enable) return;

    // 🔥 adiciona na fila
    this.buffer.push(log);

    // 🔥 dispara flush se atingir limite
    if (this.buffer.length >= this.MAX_BUFFER) {
      this.flush().catch(console.error);
    }
  }

  private async flush(): Promise<void> {
    if (this.isFlushing) return;
    if (this.buffer.length === 0) return;

    this.isFlushing = true;

    const logsToSave = [...this.buffer];
    this.buffer = [];

    try {
      // 🔹 salva no banco se Prisma estiver disponível
      if (this.prisma) {
        try {
          await this.prisma.logsRscrm.create({
            data: {
              instance: 'whatsapp',
              logs: logsToSave,
            },
          });
        } catch (err) {
          console.error('[LogsRscrm] erro no banco, reempilhando logs', err);
          this.buffer.unshift(...logsToSave); // reempilha para tentar depois
          return;
        }
      }

      // 🔹 salva no arquivo (append)
      await fs.mkdir(path.dirname(LogsRscrm.FILE_PATH), { recursive: true });
      await fs.appendFile(
        LogsRscrm.FILE_PATH,
        logsToSave.map((l) => JSON.stringify(l)).join('\n') + '\n',
        'utf-8'
      );

      console.log('[LogsRscrm] flush realizado:', logsToSave.length);
    } catch (err) {
      console.error('[LogsRscrm] erro no flush:', err);
      this.buffer.unshift(...logsToSave); // reempilha para tentar depois
    } finally {
      this.isFlushing = false;
    }
  }
}
